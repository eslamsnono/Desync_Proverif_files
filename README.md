# ProVerif models: credential recovery in the Li and Hu scheme

Models and saved outputs for the paper *Credential Recovery in IoT Authentication Schemes: Desynchronization by Replay Attack and a Verified Recovery Rule* (IEEE-ICCA'26, submitted). They analyze the resynchronization message of the scheme of Li and Hu, "A lightweight ECC-based authentication and key agreement protocol for IoT with dynamic authentication credentials", *Sensors*, vol. 24, no. 24, 2024, Sections 3.3 and 3.4, and the three-part recovery rule R1--R3 of the paper.

Tool: ProVerif 2.05. Every model was run with

    proverif <model>.pv > <model>.out.txt

Each `.out.txt` file is the complete console output of one run. To compare a new run, write it to a new file instead of overwriting the saved one.

## File names

Names follow the pattern `li2024_<scenario>_<variant>`:

- `insync`: the device is in step with the server when the attack starts (Table III of the paper), bounded to two logins;
- `lagging`: the device's final reply was lost, so it is one step behind, bounded to one recovery episode;
- `asprinted`: the server's step A2 as printed, writing state before it checks V1;
- `verifyfirst`: the server checks V1 before any state change;
- `R1`, `R1_only`, `R1_R2_R3`: which parts of the recovery rule are added;
- `refusal`: the query is a refused login of the honest device;
- `replay_unbounded`: injective agreement on the recovery credential, unbounded sessions.

## Files and the results of the paper

Eight of the original nine model/output pairs support results reported in the paper (Table IV and Section IV-E). One pair can support several rows of Table IV, since a model holds several queries, so the pairs do not map one-to-one to rows. The additional case-(d) pairs are described below. The pair `li2024_insync_nonce_control` is an extra control that is not reported in the paper: it binds the recovery message to a separate device nonce, whereas the paper's R1 results bind it to the login's As. The last column gives the working name used in the project notes.

| Model (`.pv`, output `.out.txt`) | Scenario | Query | Saved result | Bound | Working name |
|---|---|---|---|---|---|
| `li2024_insync_replay_unbounded` | in step, as printed | injective agreement between the device installing a recovery credential and the server issuing it | counterexample: one issue, two installs; non-injective agreement holds | unbounded sessions | `li2024_resync_replay` |
| `li2024_insync_nonce_control` | in step, device nonce in the message (control, not reported) | same query | agreement holds | unbounded sessions | `li2024_resync_control` |
| `li2024_insync_asprinted_refusal` | in step, A2 as printed | refused login after a replayed recovery message | reachable; the trace uses a modified V1 | 2 logins | `li2024_strict2` |
| `li2024_insync_verifyfirst_refusal` | in step, V1 verified first | refused login after a replayed recovery message; any refused second login; honest recovery | reachable (pure replay trace of Table III); honest recovery reachable | 2 logins | `li2024_strict2_conservative` |
| `li2024_insync_asprinted_R1` | in step, A2 as printed, R1 (MV bound to As) | same queries | refusal unreachable; honest recovery reachable | 2 logins | `li2024_strict2_bindingonly_asprinted` |
| `li2024_insync_verifyfirst_R1` | in step, V1 verified first, R1 | same queries | refusal unreachable; honest recovery reachable | 2 logins | `li2024_strict2_control` |
| `li2024_lagging_asprinted_refusal` | lagging device, as printed | refused login after the recorded old recovery message | reachable | 1 recovery episode | `li2024_strict_baseline_v2` |
| `li2024_lagging_R1_only` | lagging device, R1 only | refused login | reachable (SR1 replaced both server credentials) | 1 recovery episode | `li2024_strict_binding_only` |
| `li2024_lagging_R1_R2_R3` | lagging device, R1--R3, one remembered As | refused login; stale credential installed; retry, re-issue, server commit, repeated request refused | refusal unreachable; stale install unreachable; retry, re-issue and server commit reachable; repeat refused reachable | 1 recovery episode | `li2024_recovery_rule_R1_R2_R3` (earlier `li2024_strict_repair_v4_oneval`) |

`GUIDE_li2024_lagging_R1_R2_R3.md` explains the last model in detail. The `history` folder keeps two earlier variants of the rule model, which give the same verdicts and are not reported in the paper: `li2024_lagging_R1_R2_R3_separate_q` (a separate request value next to As; working name `li2024_strict_repair_v2`) and `li2024_lagging_R1_R2_R3_multi_As` (several remembered As values; working name `li2024_strict_repair_v3_as`).

## Case (d): delayed old login after a fresh retry (added with V28)

These three model/output pairs test one bounded delivery sequence for a delayed old login, a byte-for-byte retransmission control, and the R1--R3 repair. They are additional to the nine models listed above.

| Model and saved output | Scenario | Result and scope |
|---|---|---|
| `case_d/li2024_case_d_late_login_fresh_retry.pv` and `.out.txt` | The attacker delays an old valid login. The device retries with a distinct fresh `As`; the server accepts that retry, then processes the delayed login. | ProVerif finds a trace: the delayed login triggers SR1 and resets both server credential slots, the attacker drops SR1, and the device's next login is refused. A2 verifies `V1` before changing state in this model. |
| `case_d/li2024_case_d_exact_retransmission_control.pv` and `.out.txt` | The device retransmits the identical M1, including the same `As` and `V1`, in the delivery order encoded in the model. | The Login check ignores the duplicate, SR1 is not issued, and the next login is accepted in this tested order. This is a schedule-specific control. |
| `case_d/li2024_case_d_R1_R2_R3.pv` and `.out.txt` | The delayed-login sequence is tested with the pending-credential rule and the recovery response bound to its request. | ProVerif reaches the modeled retry, pending-credential installation, proof of possession and server commit. It reports no stale installation and no refusal of that modeled proof login. The model ends at commit. |

The three models use one device and one server with symbolic hash and XOR. They are scripted bounded scenarios. They do not establish the result for every delivery order, concurrent sessions, persistent storage, power failure or rollback. To rerun a model from the repository root while preserving the checked output, use, for example:

    proverif case_d/li2024_case_d_late_login_fresh_retry.pv > case_d/li2024_case_d_late_login_fresh_retry.rerun.out.txt

## Reading the results

- Queries of the form `not (event A && event B)` ask whether an undesirable combination can occur. `RESULT ... is true` means ProVerif proved it unreachable in the model. `RESULT ... is false` with "A trace has been found" means a counterexample exists.
- Queries of the form `not event E` are reachability witnesses. `is false` means the event is reachable, which is the desired answer for retry, re-issue and commit events.
- The rule model ends at the server's commit event. It does not model the login that follows the commit.

## Abstractions and limits

- One device and one server. Bounds as in the table; the two unbounded models bound only the query, not the sessions.
- Hashes are free functions. XOR is a constructor with an inverse rule, not XOR algebra; the static mask h(WT || X) therefore carries no confidentiality claim. The elliptic-curve key agreement, which plays no role in the credential update, is replaced by a hash.
- A check value h(message || key) is treated as an unforgeable tag. This is an assumption of the symbolic model.
- No storage, crash, rollback or concurrent sessions are modeled. The write order of the rule is an implementation assumption.
- Fresh values are unique in the model; an implementation must make As collisions negligible and prevent reuse after a reboot or rollback.

## Scenario of the attack (Table III of the paper)

An earlier recovery created credential R0 and the attacker recorded its recovery message. One normal login followed, so the device holds N1 and the server (N1, R0). The device sends a login, the attacker answers it with the recorded message, the device installs R0, the attacker then delivers the held-back login so the server advances to (N2, N1), and the device's next login with R0 is refused.
