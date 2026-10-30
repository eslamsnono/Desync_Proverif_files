# Guide to the Li and Hu recovery-rule model

This note explains the ProVerif model in **li2024_lagging_R1_R2_R3.pv** and its saved output in **li2024_lagging_R1_R2_R3.out.txt**.

The model tests whether a replayed, old resynchronization message can leave a sensor and server out of step, and whether the proposed three-part recovery rule prevents the modeled refusal. It represents one sensor, one server, and one bounded recovery episode.

## Files

- **li2024_lagging_R1_R2_R3.pv** — the ProVerif input model.
- **li2024_lagging_R1_R2_R3.out.txt** — output saved from the recorded ProVerif 2.05 run.
- **README.md** — the repository front page, with every model and its saved output.

The output file is a record of an earlier run. This guide does not rerun ProVerif.

## Scenario in plain language

The model follows these events:

1. The sensor and server start with credential TC0. An earlier recovery message for TC0 has been recorded by the attacker. That message is bound to the request value AsOld.
2. The sensor sends a login using TC0. The server accepts it and advances to TC1. The model drops the server's final reply, so the sensor remains at TC0 while the server has current TC1 and previous TC0.
3. The sensor tries again with a fresh request value, AsNow. The attacker replays the recorded recovery message. Because its check value was made for AsOld, the sensor rejects it and keeps TC0.
4. The sensor retries with a new request value. The server sends its pending credential again, now bound to that retry value. The sensor can accept it.
5. The sensor later proves it holds the pending credential. The server then reaches the commit event.

The model represents the attacker's ability to observe, block, replay, reorder, and inject network messages. The private stage channel only coordinates the two modeled processes. It is not a message sent over the network.

## What the three rule parts mean here

- **R1 — Bind the response to the request.** The recovery check value includes the sensor's fresh As value. The sensor checks it against the As value for its pending login. An old response carries the wrong As and is rejected.
- **R2 — Keep a credential pending.** The server keeps the new credential in a pending slot. It does not make it current until a later valid login proves that the sensor has received it.
- **R3 — Verify before changing state.** The server validates the login check before it changes the modeled credential state.

The server also remembers the As value of the last login it answered and refuses an exact repeat of that request. In this model, that remembered value is temporary state.

## Run the model

Use ProVerif 2.05. From a terminal in the directory containing the model, run:

    proverif li2024_lagging_R1_R2_R3.pv

If the ProVerif executable is not on the system path, run it using the path for your installation. The command may be named differently on Windows.

To keep a fresh run for comparison, redirect its output to a new file rather than overwriting the saved output:

    proverif li2024_lagging_R1_R2_R3.pv > rerun_li2024_lagging_R1_R2_R3.txt

## How to read the results

The model contains several queries, so one ProVerif run reports several results. This is why more than one row in the paper's results table can refer to this single model and output pair.

The saved output reports:

| Question | Saved result | Plain meaning |
|---|---|---|
| Can the modeled honest sensor's final login be refused because its credential matches neither server value? | Unreachable | No such trace was found in this bounded model. |
| Can the sensor install the credential from the recorded old recovery message? | Unreachable | The stale credential is not installed in this model. |
| Can the sensor reject the old message and retry? | Reachable | ProVerif found a trace where this happens. |
| Can the server re-issue the pending credential for the retry? | Reachable | ProVerif found a trace where it happens. |
| Can the server commit the pending credential? | Reachable | ProVerif found the server-side commit event. |
| Can the sensor install a recovery credential? | Reachable | A genuine recovery path remains possible. |
| Can the server refuse a repeated request value? | Reachable | ProVerif found the modeled duplicate-request rejection. |

Read the full query together with its result. In this file, the security queries ask whether an undesirable event can happen: ProVerif reports the negated query as **true** when that event is unreachable. The reachability queries ask whether a desired event can happen: ProVerif reports the negated query as **false** and gives a trace when it is reachable. A bare **true** or **false** is not enough to interpret a result.

The commit result means that the server reaches the modeled commit point. The model does not continue through the later normal login exchange, so it does not establish that a complete login finishes after commit.

## Limits

These results apply only to the scenario and abstractions in this file:

- The run has one sensor, one server, and one bounded recovery episode. It is not an unbounded proof of the full protocol.
- The model does not represent power loss, durable storage, atomic writes, rollback after reboot, concurrent sessions, or permanent blocking of messages.
- Hashes are symbolic free functions. The XOR operation is modeled as a constructor with an inverse rule, not with the full algebra of XOR. The model therefore makes no claim about confidentiality of credentials or the security of a real hash construction as a message authentication code.
- The elliptic-curve key agreement is abstracted because the recovery-state question does not depend on its curve arithmetic.
- The result is about the printed protocol model, not an implementation.

## For the paper's reproducibility repository

Keep this model next to its matching saved output. The paper's baseline and control results require their own model and output files too; this pair alone does not reproduce all rows of Table IV. Include the ProVerif version and the exact run command in the repository README. If these files are renamed, update the README and keep each output matched to its model.
