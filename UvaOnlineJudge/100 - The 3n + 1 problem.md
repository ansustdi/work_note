```cpp
#include <bits/stdc++.h>

using namespace std;

int solve(int n) {
  int ans = 1;
  while (n > 1) {
    if (n & 1) {
      n = n * 3 + 1;
    } else {
      n >>= 1;
    }
    ans++;
  }
  return ans;
}

int main() {
  // freopen("input.txt", "r", stdin);
  // freopen("output.txt", "w", stdout);
  ios::sync_with_stdio(false);
  cin.tie(nullptr);

  int m, n;
  while (cin >> m >> n) {
    int max_cycle = 0;
    for (int i = min(m, n); i <= max(m, n); i++) {
      max_cycle = max(max_cycle, solve(i));
    }
    cout << m << " " << n << " " << max_cycle << "\n";
  }
  return 0;
}
```