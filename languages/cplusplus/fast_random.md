# Fast Random

## Example Code

```c++
#include <cstdint>
#include <random>
using namespace std;

random_device rd;
/* The state must be seeded so that it is not everywhere zero. */
uint64_t s[2] = { (uint64_t(rd()) << 32) ^ (rd()),
    (uint64_t(rd()) << 32) ^ (rd()) };
uint64_t curRand;
uint8_t bit = 63;

uint64_t xorshift128plus(void) {
    uint64_t x = s[0];
    uint64_t const y = s[1];
    s[0] = y;
    x ^= x << 23; // a
    s[1] = x ^ y ^ (x >> 17) ^ (y >> 26); // b, c
    return s[1] + y;
}

bool randBool()
{
    if(bit >= 63)
    {
        curRand = xorshift128plus();
        bit = 0;
        return curRand & 1;
    }
    else
    {
        bit++;
        return curRand & (1<<bit);
    }
}
```

---

Reference from *[What is performance-wise the best way to generate random bools?](https://stackoverflow.com/questions/35358501/what-is-performance-wise-the-best-way-to-generate-random-bools)* by Serge Rogatch