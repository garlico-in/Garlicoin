### TestGen ###

Utilities to generate test vectors for the data-driven Garlicoin tests.

Usage: 

    PYTHONPATH=../../test/functional/test_framework ./gen_key_io_test_vectors.py valid 50 > ../../src/test/data/base58_keys_valid.json
    PYTHONPATH=../../test/functional/test_framework ./gen_key_io_test_vectors.py invalid 50 > ../../src/test/data/base58_keys_invalid.json