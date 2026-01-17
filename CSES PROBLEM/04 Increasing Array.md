
```cpp
#include <bits/stdc++.h>

using namespace std;

int main() {
  int n;
  scanf("%d", &n);
  long long ans = 0;
  int prev;
  scanf("%d", &prev);
  for (int i = 1; i < n; i++) {
    int current;
    scanf("%d", &current);
    ans += max(0, prev - current);
    prev = max(prev, current);
  }
  printf("%lld\n", ans);
  return 0;
}
```