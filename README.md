parallel-murmur3
----------------

Three AVX2 implementations of the Murmur3 hash functions.

- `parallel`: Computing the hash values of 8 keys in parallel.
- `parallel_multiseed`: Computing N hash values for each of 8 keys
  in parallel. Each of the N hash values for a key will be computed
  using a different key.
- `scalar`: Computing the hash value of a single key.

All of the above implementations have the same constraints on
keys:

- A constant size (so not e.g. on strings)
- A size that's an multiple of 4 bytes.

These are very special purpose implementations, and will not be
of any interest in most programs. Use these functions if:

- You have an application that can receive and process inputs
  in batches such that you usually have 8 keys to process at once.
  Doesn't need to be full batches of 8, but the break-even point
  vs. a fast scalar hash is probably around batches of 3-4 keys.
- The keys are fairly small. Many scalar hash functions benefit
  from having large key sizes. These implementations don't. The
  break-even point should be at around 64 bytes.
- You can arrange for your hash keys to be in a column-major
  order without too much pain.

If the above points aren't true for you application, you're
probably better off using e.g. [CityHash](https://github.com/google/cityhash).

In particular the `scalar` implementation is only included as a fallback, for
programs that generally use the parallel versions, but have some
exceptional cases that need identical hash codes for individual
key. There's not a lot of point in just using the `scalar` implementation.

DATA LAYOUT
-----------

For the parallel implementations the keys should be laid out
adjacent to each other, in column-major order. That is, the first
word in "keys" should be the first word of the first key. The
second word of "keys should be the first word of the second
key. And so on:

```
  key1[0] key2[0] ... key7[0]
  key2[1] key2[1] ... key7[1]
  ...
  key1[SizeWords-1] key2[SizeWords-1] ... key7[SizeWords-1]
```

EXAMPLES
--------

Assume the following definitions:

```c++
  static const uint32_t KEY_LENGTH = 3;

   static uint32_t rows[][KEY_LENGTH] = {
      {1, 2, 3},
      {4, 5, 6},
      {7, 8, 9},
      {10, 11, 12},
      {13, 14, 15},
      {16, 17, 18},
      {19, 20, 21},
      {22, 23, 24},
  };

  static uint32_t cols[][8] = {
      {1, 4, 7, 10, 13, 16, 19, 22},
      {2, 5, 8, 11, 14, 17, 20, 23},
      {3, 6, 9, 12, 15, 18, 21, 24},
  };

  static uint32_t seeds[] = { 0x3afc8e77, 0x924f408d };
  static const uint32_t seed_count = sizeof(seeds) / sizeof(uint32_t);
```

The three implementations could be used as follows to compute
hash values for each of the key / seed combinations:

```c++
  void example_scalar() {
      for (int s = 0; s < seed_count; ++s) {
          for (int i = 0; i < 8; ++i) {
              uint32_t* row = rows[i];
              uint32_t res = murmur3<3>::scalar(row, seeds[s]);
              printf("seed=%08x, key={%u,%u,%u}, hash=%u\n",
                     seeds[s], row[0], row[1], row[2], res);
          }
      }
  }

  void example_parallel() {
      for (int s = 0; s < seed_count; ++s) {
          __m256i hash = murmur3<3>::parallel(cols[0], seeds[s]);
          uint32_t res[8];
          _mm256_storeu_si256((__m256i*) res, hash);
          for (int i = 0; i < 8; ++i) {
              printf("seed=%08x, key={%u,%u,%u}, hash=%u\n",
                     seeds[s], cols[0][i], cols[1][i], cols[2][i],
                     res[i]);
          }
      }
  }

  void example_parallel_multiseed() {
      __m256i hash[seed_count];
      murmur3<3>::parallel_multiseed<seed_count>(cols[0], seeds, hash);
      for (int s = 0; s < seed_count; ++s) {
          uint32_t res[8];
          _mm256_storeu_si256((__m256i*) res, hash[s]);
          for (int i = 0; i < 8; ++i) {
              printf("seed=%08x, key={%u,%u,%u}, hash=%u\n",
                     seeds[s], cols[0][i], cols[1][i], cols[2][i],
                     res[i]);
          }
      }
  }
```