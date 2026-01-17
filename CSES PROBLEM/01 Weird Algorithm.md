```cpp
#include <bits/stdc++.h>

using namespace std;

int main() {
  long long n;
  scanf("%lld", & n);
  printf("%lld", n);
  while (n > 1) {
    if (n & 1) {
      n = 3 * n + 1;
    } else {
      n >>= 1;
    }
    printf(" %lld", n);
  }
  return 0;
}
```