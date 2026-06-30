# Benchmark matrix — compare

Mean ± 95 % bootstrap CI per cell, plus paired McNemar between approaches within each generator (success definition shown).

## Success rate — Strict

| Generator | structured | unstructured | McNemar p (paired) |
| --- | --- | --- | --- |
| claude-sonnet-4-6 | 0.200 [0.161, 0.242] | 0.225 [0.183, 0.267] | 0.380 (n=360) |
| gpt-5.5 | 0.361 [0.314, 0.411] | 0.417 [0.367, 0.469] | 0.103 (n=360) |
| grok-build-0.1 | 0.411 [0.361, 0.461] | 0.294 [0.250, 0.344] | 0.000 (n=360) |

## Success rate — Lenient

| Generator | structured | unstructured | McNemar p (paired) |
| --- | --- | --- | --- |
| claude-sonnet-4-6 | 0.200 [0.161, 0.242] | 0.225 [0.183, 0.267] | 0.000 (n=360) |
| gpt-5.5 | 0.361 [0.314, 0.411] | 0.417 [0.367, 0.469] | 0.000 (n=360) |
| grok-build-0.1 | 0.411 [0.361, 0.461] | 0.294 [0.250, 0.344] | 0.532 (n=360) |

## Latency (ms)

| Generator | structured | unstructured |
| --- | --- | --- |
| claude-sonnet-4-6 | 8652 [8264, 9036] | 7577 [7120, 8129] |
| gpt-5.5 | 6358 [6014, 6704] | 5360 [5096, 5631] |
| grok-build-0.1 | 7920 [7430, 8447] | 6374 [6040, 6716] |

## Lexical similarity

| Generator | structured | unstructured |
| --- | --- | --- |
| claude-sonnet-4-6 | 0.376 [0.369, 0.384] | 0.412 [0.405, 0.418] |
| gpt-5.5 | 0.435 [0.429, 0.441] | 0.480 [0.475, 0.485] |
| grok-build-0.1 | 0.438 [0.431, 0.444] | 0.446 [0.440, 0.451] |

## Estimated cost (USD)

| Generator | structured | unstructured |
| --- | --- | --- |
| claude-sonnet-4-6 | 0.00669 [0.00637, 0.00700] | 0.00456 [0.00434, 0.00477] |
| gpt-5.5 | 0.00892 [0.00845, 0.00939] | 0.01405 [0.01371, 0.01436] |
| grok-build-0.1 | 0.00392 [0.00375, 0.00410] | 0.00622 [0.00608, 0.00636] |
