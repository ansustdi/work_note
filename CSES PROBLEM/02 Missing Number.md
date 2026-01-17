> [!TIP] > This uses frequency array  to find the answer in $O(n)$  space.

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
  int n;
  scanf("%d", & n);
  vector < int > v(n, 0);
  for (int i = 0; i < n - 1; i++) {
    int x;
    scanf("%d", & x);
    v[x - 1]++;
  }
  for (int i = 0; i < n; i++) {
    if (v[i] == 0) {
      printf("%d\n", i + 1);
      break;
    }
  }
  return 0;
}
```


> [!TIP] > This uses the formula $Sum = \frac{n(n+1)}{2}$ to find the answer in $O(1)$ extra space.

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
  int n;
  scanf("%d", &n);
  long long ans = 0;
  for (int i = 0; i < n - 1; i++) {
    long long x;
    scanf("%lld", &x);
    ans += x;
  }
  long long sum = (long long)n * (n + 1) / 2;
  printf("%lld\n", sum - ans);
  return 0;
}
```