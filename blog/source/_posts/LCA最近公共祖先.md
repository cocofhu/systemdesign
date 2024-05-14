---
title: LCA最近公共祖先
date: 2024-05-14 23:49:34
---


### 一、常用模板

#### 1、树上倍增
```c++
// https://www.luogu.com.cn/problem/P3379
const int maxn = 500010;
using ll = long long int;
struct Edge {
    int next, to;
} edges[maxn * 2];
int heads[maxn], pos = 0;

void addEdge(int a, int b) {
    edges[++pos].next = heads[a];
    edges[pos].to = b;
    heads[a] = pos;
}
int st[maxn][20];
int depth[maxn];
void dfs(int u, int fa){
    depth[u] = fa == -1 ? 0 : depth[fa] + 1;
    st[u][0] = fa;
    for(int i = 1; ~st[u][i-1] && i < 20; ++i){
        st[u][i] = st[st[u][i-1]][i-1];
    }
    for(int i = heads[u]; ~i; i = edges[i].next){
        int v = edges[i].to;
        if(fa == v) continue;
        dfs(v, u);
    }
}
int lca(int u, int v){
    if(depth[u] < depth[v]) return lca(v, u);
    for(int i = 19; ~i; --i){
        if(~st[u][i] && depth[st[u][i]] >= depth[v]) u = st[u][i];
    }
    if(u == v) return u;
    for(int i = 19; ~i; --i){
        if(st[u][i] != st[v][i]) u = st[u][i], v = st[v][i];
    }
    return st[u][0];
}
int main(){
    int n, m, s, a, b;
    memset(heads, -1, sizeof(heads));
    memset(st, -1, sizeof(st));
    scanf("%d%d%d", &n, &m, &s);
    for(int i = 1; i < n; ++i){
        scanf("%d%d", &b, &a);
        if(a == b) continue;
        addEdge(a, b);
        addEdge(b, a);
    }
    dfs(s, -1);
    for(int i = 0; i < m; ++i){
        scanf("%d%d", &b, &a);
        printf("%d\n", lca(a, b));
    }
    return 0;
}

```

