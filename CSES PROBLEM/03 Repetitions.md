```cpp
#include <bits/stdc++.h>
using namespace std;
const int mxn = 1e6 + 1;
char s[mxn];
int main() {

  scanf("%s", s);
  int n = strlen(s);
  int cur = 1;
  int ans = 0;
  for (int i = 1; i < n; i++) {
    if (s[i] == s[i - 1]) {
      cur++;
    } else {
      ans = max(ans, cur);
      cur = 1;
    }
  }
  printf("%d\n", max(ans, cur));
  return 0;
}
```