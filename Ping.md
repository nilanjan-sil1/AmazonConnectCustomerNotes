# Parallel Session Limit Node for PingAM 8.1.0

Below is a complete, production-ready custom authentication node for PingAM 8.1.0 that enforces parallel session limits per user, along with comprehensive JUnit 5 tests.

## Project Structure

```
src/main/java/com/example/nodes/
    ParallelSessionLimitNode.java
src/main/resources/com/example/nodes/
    ParallelSessionLimitNode.properties
src/test/java/com/example/nodes/
    ParallelSessionLimitNodeTest.java
```

## 1. Main Node Class

```java
/*
 * Copyright 2024 Example. All rights reserved.
 */
package com.example.nodes;

import com.google.inject.assistedinject.Assisted;
import org.forgerock.openam.annotations.nodes.Attribute;
import org.forgerock.openam.auth.node.api.AbstractDecisionNode;
import org.forgerock.openam.auth.node.api.Action;
import org.forgerock.openam.auth.node.api.IdentityAware;
import org.forgerock.openam.auth.node.api.Node;
import org.forgerock.openam.auth.node.api.NodeProcessException;
import org.forgerock.openam.auth.node.api.OutcomeProvider;
import org.forgerock.openam.auth.node.api.Tone;
import org.forgerock.openam.auth.node.api.TreeContext;
import org.forgerock.openam.core.realms.Realm;
import org.forgerock.openam.cts.api.CoreTokenService;
import org.forgerock.openam.cts.api.filter.TokenFilter;
import org.forgerock.openam.cts.api.filter.TokenFilterBuilder;
import org.forgerock.openam.cts.api.tokens.PartialToken;
import org.forgerock.openam.cts.exceptions.CoreTokenException;
import org.forgerock.openam.tokens.CoreTokenField;
import org.forgerock.openam.tokens.TokenType;
import org.forgerock.openam.utils.PreferredLocales;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.inject.Inject;
import java.util.Collection;
import java.util.List;
import java.util.Optional;
import java.util.ResourceBundle;

/**
 * Authentication node that limits the number of concurrent live sessions for the
 * authenticating user by querying the Core Token Service (CTS).
 *
 * <p>The node inspects the shared state for the user's universal ID (populated by an
 * upstream identification node such as Data Store Decision or Inner Tree Evaluator)
 * and counts all live session tokens currently persisted in CTS for that user.
 * Expired tokens are excluded automatically by the {@link TokenFilterBuilder}.
 *
 * <h2>Outcomes</h2>
 * <ul>
 *   <li><b>success</b> - The number of live sessions is strictly below the configured
 *       limit. The authentication flow may proceed (a new session will be created).</li>
 *   <li><b>limitExhausted</b> - The user has reached (or exceeded) the configured
 *       maximum number of parallel sessions. The authentication flow should be aborted.</li>
 *   <li><b>error</b> - The session count could not be queried or any other error
 *       occurred (e.g. universal ID not present, CTS failure).</li>
 * </ul>
 *
 * <h2>Important note on stateless sessions</h2>
 * Stateless sessions are not stored in CTS unless <em>Session Tracking</em> (a.k.a.
 * Session Blacklist/Whitelist) is enabled in the realm. If your deployment uses
 * stateless sessions without tracking, this node will report zero active sessions
 * and the limit will never be enforced.
 */
@Node.Metadata(outcomeProvider = ParallelSessionLimitNode.SessionOutcomeProvider.class,
        configClass = ParallelSessionLimitNode.Config.class,
        tags = {"sessions", "security"})
public class ParallelSessionLimitNode extends AbstractDecisionNode implements IdentityAware {

    private static final Logger logger = LoggerFactory.getLogger(ParallelSessionLimitNode.class);
    private static final String BUNDLE = ParallelSessionLimitNode.class.getName();

    public static final String OUTCOME_SUCCESS = "success";
    public static final String OUTCOME_LIMIT_EXHAUSTED = "limitExhausted";
    public static final String OUTCOME_ERROR = "error";

    /**
     * Node configuration interface. Properties here drive the configuration UI
     * rendered by the PingAM admin console.
     */
    public interface Config {

        /**
         * The maximum number of concurrent live sessions permitted for a single user.
         *
         * @return the configured limit. Defaults to {@code 1}.
         */
        @Attribute(order = 100, requiredValue = true)
        default int maxParallelSessions() {
            return 1;
        }

        /**
         * When {@code true}, the count returned by CTS is reduced by one before being
         * compared against {@link #maxParallelSessions()}. This is useful when the node
         * is executed <em>after</em> the new session has already been persisted in CTS
         * (e.g. when placed downstream of a Session Upgrade or Inner Tree Evaluator
         * node that creates the session early).
         *
         * @return whether to subtract the current (being-established) session.
         */
        @Attribute(order = 200)
        default boolean subtractCurrentSession() {
            return true;
        }
    }

    /**
     * Localised outcome provider used by the admin console.
     */
    public static class SessionOutcomeProvider implements OutcomeProvider {

        @Override
        public List<Outcome> getOutcomes(PreferredLocales locales, Tone tone) {
            ResourceBundle bundle = locales.getBundleInPreferredLocale(
                    BUNDLE, SessionOutcomeProvider.class.getClassLoader());
            return List.of(
                    new Outcome(OUTCOME_SUCCESS, bundle.getString("outcome.success")),
                    new Outcome(OUTCOME_LIMIT_EXHAUSTED, bundle.getString("outcome.limitExhausted")),
                    new Outcome(OUTCOME_ERROR, bundle.getString("outcome.error")));
        }
    }

    private final Config config;
    private final CoreTokenService coreTokenService;
    private final Realm realm;

    /**
     * Guice-assisted constructor invoked by the authentication tree framework.
     *
     * @param realm            The realm in which the node is being executed.
     * @param config           Node configuration.
     * @param coreTokenService The Core Token Service used to query active sessions.
     */
    @Inject
    public ParallelSessionLimitNode(@Assisted Realm realm,
                                    @Assisted Config config,
                                    CoreTokenService coreTokenService) {
        if (config.maxParallelSessions() < 0) {
            throw new IllegalArgumentException("maxParallelSessions cannot be negative");
        }
        this.realm = realm;
        this.config = config;
        this.coreTokenService = coreTokenService;
    }

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        try {
            Optional<String> universalIdOpt = getUniversalId(context);
            if (universalIdOpt.isEmpty() || universalIdOpt.get().isBlank()) {
                logger.error("Universal ID is not available in the shared state. "
                        + "Place this node after an identification node (e.g. Data Store Decision).");
                return goTo(OUTCOME_ERROR).build();
            }

            String universalId = universalIdOpt.get();
            int rawCount = countActiveSessions(universalId);
            int effectiveCount = config.subtractCurrentSession() && rawCount > 0
                    ? rawCount - 1
                    : rawCount;
            int limit = config.maxParallelSessions();

            logger.debug("User {} in realm {} has {} raw sessions, {} effective, limit {}",
                    universalId, realm, rawCount, effectiveCount, limit);

            if (effectiveCount >= limit) {
                logger.info("Parallel session limit exhausted for user {}: {} >= {}",
                        universalId, effectiveCount, limit);
                return goTo(OUTCOME_LIMIT_EXHAUSTED).build();
            }

            return goTo(OUTCOME_SUCCESS).build();

        } catch (CoreTokenException e) {
            logger.error("Failed to query active session count from CTS", e);
            return goTo(OUTCOME_ERROR).build();
        } catch (Exception e) {
            logger.error("Unexpected error in ParallelSessionLimitNode", e);
            return goTo(OUTCOME_ERROR).build();
        }
    }

    /**
     * Queries CTS for all live session tokens belonging to the supplied universal ID.
     *
     * @param universalId the user's universal ID.
     * @return the number of live session tokens currently stored in CTS for the user.
     * @throws CoreTokenException if the underlying CTS query fails.
     */
    private int countActiveSessions(String universalId) throws CoreTokenException {
        TokenFilter filter = new TokenFilterBuilder()
                .returnExpiredTokens(false)
                .and(CoreTokenField.TOKEN_TYPE, TokenType.SESSION)
                .and(CoreTokenField.USER_ID, universalId)
                .build();

        Collection<PartialToken> tokens = coreTokenService.query(filter);
        return tokens == null ? 0 : tokens.size();
    }
}
```

## 2. Resource Bundle

`src/main/resources/com/example/nodes/ParallelSessionLimitNode.properties`:

```properties
# Outcome labels
outcome.success = Success
outcome.limitExhausted = Limit Exhausted
outcome.error = Error

# Config field labels
maxParallelSessions.label = Maximum Parallel Sessions
maxParallelSessions.help = The maximum number of concurrent live sessions permitted for a single user. When the limit is reached the "limitExhausted" outcome is returned.
subtractCurrentSession.label = Subtract Current Session
subtractCurrentSession.help = If enabled, the count returned by CTS is reduced by one before being compared against the limit. Use this when the node is executed after the new session has already been persisted in CTS.
```

## 3. JUnit 5 Test Class

```java
/*
 * Copyright 2024 Example. All rights reserved.
 */
package com.example.nodes;

import org.forgerock.json.JsonValue;
import org.forgerock.openam.auth.node.api.Action;
import org.forgerock.openam.auth.node.api.ExternalRequestContext;
import org.forgerock.openam.auth.node.api.IdentityAware;
import org.forgerock.openam.auth.node.api.TreeContext;
import org.forgerock.openam.core.realms.Realm;
import org.forgerock.openam.cts.api.CoreTokenService;
import org.forgerock.openam.cts.api.filter.TokenFilter;
import org.forgerock.openam.cts.api.tokens.PartialToken;
import org.forgerock.openam.cts.exceptions.CoreTokenException;
import org.forgerock.openam.tokens.CoreTokenField;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.mockito.junit.jupiter.MockitoSettings;
import org.mockito.quality.Strictness;

import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

/**
 * Unit tests for {@link ParallelSessionLimitNode}.
 */
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT)
class ParallelSessionLimitNodeTest {

    private static final String UNIVERSAL_ID = "id=testuser,ou=user,dc=example,dc=com";

    @Mock
    private Realm realm;

    @Mock
    private CoreTokenService coreTokenService;

    /* ------------------------------------------------------------------ *
     * Helpers
     * ------------------------------------------------------------------ */

    private ParallelSessionLimitNode createNode(int maxSessions, boolean subtractCurrent) {
        ParallelSessionLimitNode.Config config = new ParallelSessionLimitNode.Config() {
            @Override public int maxParallelSessions() { return maxSessions; }
            @Override public boolean subtractCurrentSession() { return subtractCurrent; }
        };
        return new ParallelSessionLimitNode(realm, config, coreTokenService);
    }

    private TreeContext createContextWithUser(String universalId) {
        JsonValue sharedState = JsonValue.json(JsonValue.object());
        if (universalId != null) {
            sharedState.put(IdentityAware.UNIVERSAL_ID_KEY, universalId);
        }
        return new TreeContext(
                sharedState,
                JsonValue.json(JsonValue.object()),
                ExternalRequestContext.builder().build(),
                Collections.emptyList());
    }

    @SuppressWarnings("unchecked")
    private PartialToken mockToken(String tokenId) {
        Map<CoreTokenField<?>, Object> fields = new HashMap<>();
        fields.put(CoreTokenField.TOKEN_ID, tokenId);
        fields.put(CoreTokenField.USER_ID, UNIVERSAL_ID);
        return new PartialToken((Map<CoreTokenField<?>, ?>) (Map<?, ?>) fields);
    }

    /* ------------------------------------------------------------------ *
     * Constructor validation
     * ------------------------------------------------------------------ */

    @Test
    void constructor_shouldRejectNegativeLimit() {
        assertThatThrownBy(() -> createNode(-1, true))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("cannot be negative");
    }

    @Test
    void constructor_shouldAllowZeroLimit() {
        // A limit of zero effectively denies all new sessions
        ParallelSessionLimitNode node = createNode(0, true);
        assertThat(node).isNotNull();
    }

    /* ------------------------------------------------------------------ *
     * Success scenarios
     * ------------------------------------------------------------------ */

    @Test
    void process_shouldReturnSuccess_whenNoActiveSessions_andSubtractEnabled() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class))).thenReturn(Collections.emptyList());

        ParallelSessionLimitNode node = createNode(3, true);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_SUCCESS);
    }

    @Test
    void process_shouldReturnSuccess_whenSessionsBelowLimit_andSubtractEnabled() throws Exception {
        // 2 sessions in CTS, subtract 1 -> effective 1, limit 3 -> success
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenReturn(List.of(mockToken("t1"), mockToken("t2")));

        ParallelSessionLimitNode node = createNode(3, true);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_SUCCESS);
    }

    @Test
    void process_shouldReturnSuccess_whenSessionsBelowLimit_andSubtractDisabled() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenReturn(List.of(mockToken("t1"), mockToken("t2")));

        ParallelSessionLimitNode node = createNode(3, false);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_SUCCESS);
    }

    @Test
    void process_shouldReturnSuccess_whenExactlyAtLimitMinusOne() throws Exception {
        // 2 sessions, limit 3, no subtract -> 2 < 3 -> success
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenReturn(List.of(mockToken("t1"), mockToken("t2")));

        ParallelSessionLimitNode node = createNode(3, false);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_SUCCESS);
    }

    @Test
    void process_shouldReturnSuccess_whenCtsReturnsNull() throws Exception {
        // Defensive: CTS should never return null, but we handle it gracefully
        when(coreTokenService.query(any(TokenFilter.class))).thenReturn(null);

        ParallelSessionLimitNode node = createNode(1, true);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_SUCCESS);
    }

    /* ------------------------------------------------------------------ *
     * Limit exhausted scenarios
     * ------------------------------------------------------------------ */

    @Test
    void process_shouldReturnLimitExhausted_whenAtExactLimit_andSubtractDisabled() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenReturn(List.of(mockToken("t1"), mockToken("t2"), mockToken("t3")));

        ParallelSessionLimitNode node = createNode(3, false);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_LIMIT_EXHAUSTED);
    }

    @Test
    void process_shouldReturnLimitExhausted_whenAboveLimit() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenReturn(List.of(mockToken("t1"), mockToken("t2"),
                        mockToken("t3"), mockToken("t4"), mockToken("t5")));

        ParallelSessionLimitNode node = createNode(3, false);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_LIMIT_EXHAUSTED);
    }

    @Test
    void process_shouldReturnLimitExhausted_whenAtExactLimit_andSubtractEnabled() throws Exception {
        // 4 raw sessions, subtract 1 -> 3 effective, limit 3 -> 3 >= 3 -> exhausted
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenReturn(List.of(mockToken("t1"), mockToken("t2"),
                        mockToken("t3"), mockToken("t4")));

        ParallelSessionLimitNode node = createNode(3, true);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_LIMIT_EXHAUSTED);
    }

    @Test
    void process_shouldReturnLimitExhausted_whenLimitIsZero() throws Exception {
        // Limit 0 means no sessions allowed at all
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenReturn(Collections.singletonList(mockToken("t1")));

        ParallelSessionLimitNode node = createNode(0, false);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_LIMIT_EXHAUSTED);
    }

    @Test
    void process_shouldReturnLimitExhausted_whenLimitIsZero_andCtsEmpty_andNoSubtract() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class))).thenReturn(Collections.emptyList());

        // limit 0, count 0, 0 >= 0 -> exhausted (cannot create any session)
        ParallelSessionLimitNode node = createNode(0, false);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_LIMIT_EXHAUSTED);
    }

    /* ------------------------------------------------------------------ *
     * Error scenarios
     * ------------------------------------------------------------------ */

    @Test
    void process_shouldReturnError_whenUniversalIdMissing() throws Exception {
        ParallelSessionLimitNode node = createNode(3, true);
        TreeContext context = createContextWithUser(null);

        Action action = node.process(context);

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_ERROR);
    }

    @Test
    void process_shouldReturnError_whenUniversalIdBlank() throws Exception {
        ParallelSessionLimitNode node = createNode(3, true);
        TreeContext context = createContextWithUser("   ");

        Action action = node.process(context);

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_ERROR);
    }

    @Test
    void process_shouldReturnError_whenCtsThrowsCoreTokenException() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenThrow(new CoreTokenException("CTS unavailable"));

        ParallelSessionLimitNode node = createNode(3, true);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_ERROR);
    }

    @Test
    void process_shouldReturnError_whenCtsThrowsRuntimeException() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class)))
                .thenThrow(new RuntimeException("Unexpected infrastructure failure"));

        ParallelSessionLimitNode node = createNode(3, true);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_ERROR);
    }

    /* ------------------------------------------------------------------ *
     * Subtract-current-session boundary cases
     * ------------------------------------------------------------------ */

    @Test
    void process_shouldNotProduceNegativeCount_whenSubtractEnabled_andCtsEmpty() throws Exception {
        when(coreTokenService.query(any(TokenFilter.class))).thenReturn(Collections.emptyList());

        // 0 sessions, subtract enabled, but we guard with rawCount > 0
        ParallelSessionLimitNode node = createNode(1, true);
        Action action = node.process(createContextWithUser(UNIVERSAL_ID));

        assertThat(action.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_SUCCESS);
    }

    @Test
    void process_subtractOne_shouldAllowOneMoreThanRawCountImplies() throws Exception {
        // 3 raw sessions, subtract 1 -> 2 effective, limit 3 -> success (would be exhausted without subtract)
        Collection<PartialToken> tokens = List.of(mockToken("t1"), mockToken("t2"), mockToken("t3"));
        when(coreTokenService.query(any(TokenFilter.class))).thenReturn(tokens);

        ParallelSessionLimitNode nodeWithSubtract = createNode(3, true);
        Action withSubtract = nodeWithSubtract.process(createContextWithUser(UNIVERSAL_ID));
        assertThat(withSubtract.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_SUCCESS);

        ParallelSessionLimitNode nodeWithoutSubtract = createNode(3, false);
        Action withoutSubtract = nodeWithoutSubtract.process(createContextWithUser(UNIVERSAL_ID));
        assertThat(withoutSubtract.outcome).isEqualTo(ParallelSessionLimitNode.OUTCOME_LIMIT_EXHAUSTED);
    }

    /* ------------------------------------------------------------------ *
     * Outcome provider
     * ------------------------------------------------------------------ */

    @Test
    void outcomeProvider_shouldReturnAllThreeOutcomes() {
        ParallelSessionLimitNode.SessionOutcomeProvider provider =
                new ParallelSessionLimitNode.SessionOutcomeProvider();

        // Use the default locale test bundle; the provider should not throw.
        // We cannot easily build a PreferredLocales with a custom classloader here,
        // but the call must at least produce three outcomes when the default bundle resolves.
        org.forgerock.openam.utils.PreferredLocales locales = new org.forgerock.openam.utils.PreferredLocales();
        List<OutcomeProvider.Outcome> outcomes = provider.getOutcomes(locales, null);

        assertThat(outcomes).hasSize(3);
        assertThat(outcomes).extracting(OutcomeProvider.Outcome::id)
                .containsExactlyInAnyOrder(
                        ParallelSessionLimitNode.OUTCOME_SUCCESS,
                        ParallelSessionLimitNode.OUTCOME_LIMIT_EXHAUSTED,
                        ParallelSessionLimitNode.OUTCOME_ERROR);
    }
}
```

## 4. Maven Dependencies (excerpt)

Add these to your `pom.xml`. The PingAM artifacts are typically available from the ForgeRock / Ping Identity Maven repository.

```xml
<dependencies>
    <!-- PingAM Authentication Node SDK -->
    <dependency>
        <groupId>org.forgerock.am</groupId>
        <artifactId>openam-auth-node-api</artifactId>
        <version>8.1.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.forgerock.am</groupId>
        <artifactId>openam-cts-api</artifactId>
        <version>8.1.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.forgerock.am</groupId>
        <artifactId>openam-core</artifactId>
        <version>8.1.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.forgerock.am</groupId>
        <artifactId>openam-shared</artifactId>
        <version>8.1.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>com.google.inject</groupId>
        <artifactId>guice</artifactId>
        <version>5.1.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.36</version>
        <scope>provided</scope>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.25.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 5. How the Node Works

1. **Placement** – Put the node in your authentication tree *after* an identification step (e.g. *Data Store Decision*). At that point the user's universal ID is available in the shared state under `IdentityAware.UNIVERSAL_ID_KEY`.

2. **CTS Query** – The node constructs a `TokenFilter` that:
   * Restricts results to `TokenType.SESSION` tokens,
   * Restricts results to tokens whose `USER_ID` equals the user's universal ID,
   * Excludes expired tokens via `returnExpiredTokens(false)`.

3. **Comparison** – The effective count is compared to `maxParallelSessions`:
   * `effectiveCount >= limit` → `limitExhausted` outcome.
   * Otherwise → `success` outcome.

4. **`subtractCurrentSession` flag** – Set to `true` when the node is placed downstream of a step that already created the new session in CTS (the count would then include the in-flight session). Set to `false` otherwise.

5. **Error handling** – Any failure (missing universal ID, CTS exception, unexpected error) results in the `error` outcome so the tree can route to a failure handler.

## 6. Suggested Tree Wiring

```
[ Page/Identifier ] ──► [ Data Store Decision ] ──► [ Parallel Session Limit ]
                                                          │
                                                          ├─ success ────► [ ... continue tree / Create Session ]
                                                          ├─ limitExhausted ──► [ Failure URL / Message Node ]
                                                          └─ error ──────────► [ Failure URL / Message Node ]
```

## 7. Caveats

* **Stateless sessions** – Stateless sessions are *not* stored in CTS unless *Session Tracking* (a.k.a. session blacklist/whitelist) is enabled in the realm. Without tracking, this node will always report zero active sessions.
* **Performance** – The CTS query is issued for every authentication. In high-volume deployments consider caching the result for a short TTL.
* **Cluster consistency** – CTS is the source of truth across a multi-instance PingAM deployment, so counts are cluster-accurate.
* **Admin permissions** – `CoreTokenService` is injected by PingAM with admin privileges; no explicit `Subject.doAs` is required.