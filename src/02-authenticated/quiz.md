# Concept-Check: Authentication

This arc added the machinery a *private* API demands: a login handshake that yields a session cookie, a token-bucket limiter in front of every request, and the run ledger that lets a pipeline remember what it did. This check pins down the parts that are easy to get *almost* right — how the cookie is forwarded, why a `503` at login is not the same as a `401`, the token-bucket arithmetic and the paused clock that tests it, and the one rule that makes the ledger correct: a failed run never advances the watermark.

Three of these are **Tracing** questions: read the program, decide whether it compiles, and if it does, predict its exact output before revealing the answer. Predicting is the point — a wrong prediction with a good explanation teaches more than a lucky guess.

{{#quiz ../../quizzes/02-authenticated.toml}}
