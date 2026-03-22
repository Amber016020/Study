InlineDataService

# 正規化

# user list ⇒

1. 因為我現在是把user的密碼用加密的形式寫在application.properties，另外在securityConfig.java裡，把application.properties一個一個讀出來，然後放進userRole哩，可以應用在filter，但是這樣如果要加入新的使用者，就需要改程式，我目前也不太想改放在vault，目前的訴求是希望可以只改application.properties就能新增使用者，請幫我想想可以怎麼寫會更好?

**✅ 方案一：用 `@ConfigurationProperties` 綁定 users（推薦）**

```jsx
app.security.users[0].username=user1
app.security.users[0].password={noop}1234
app.security.users[0].roles=USER

app.security.users[1].username=admin
app.security.users[1].password={noop}admin123
app.security.users[1].roles=ADMIN

2️⃣ 建一個設定 class
@Configuration
@ConfigurationProperties(prefix = "app.security")
@Getter
@Setter
public class SecurityProperties {
    private List<UserConfig> users;

    @Getter
    @Setter
    public static class UserConfig {
        private String username;
        private String password;
        private List<String> roles;
    }
}

3️⃣ 在 SecurityConfig 使用
@Bean
public UserDetailsService userDetailsService(SecurityProperties properties) {
    List<UserDetails> users = properties.getUsers().stream()
        .map(u -> User.withUsername(u.getUsername())
                .password(u.getPassword())
                .roles(u.getRoles().toArray(new String[0]))
                .build())
        .toList();

    return new InMemoryUserDetailsManager(users);
}
```

**✅ 方案三：用 JSON / 外部檔（進階一點）**

## application.properties

```
app.security.config-file=users.json
```

## users.json

```json
[
  { "username":"user1", "password":"{noop}1234", "roles": ["USER"] },
  { "username":"admin", "password":"{noop}admin", "roles": ["ADMIN"] }
]
```

# ✅ Step 1：application.properties

```
app.security.user-config-path=users.json
```

---

# ✅ Step 2：users.json（重點）

```json
[
  {
    "username":"user1",
    "password":"{bcrypt}$2a$10$abc...",
    "roles": ["USER"]
  },
  {
    "username":"admin",
    "password":"{bcrypt}$2a$10$xyz...",
    "roles": ["ADMIN"]
  }
]
```

👉 密碼建議先用程式 encode 好再貼進來

---

# ✅ Step 3：建立 DTO

```java
@Getter
@Setter
public class UserConfig {
    private String username;
    private String password;
    private List<Strinjsg> roles;
}
```

---

# ✅ Step 4：讀 JSON 的 Service（核心）

👉 這是整個方案的靈魂

```java
@Service
public class JsonUserDetailsService implements UserDetailsService {

    private final List<UserDetails> users;

    public JsonUserDetailsService(
            @Value("${app.security.user-config-path}") String path
    ) throws IOException {

        ObjectMapper mapper = new ObjectMapper();

        InputStream is = new ClassPathResource(path).getInputStream();

        List<UserConfig> configs = Arrays.asList(
                mapper.readValue(is, UserConfig[].class)
        );

        this.users = configs.stream()
                .map(u -> User.withUsername(u.getUsername())
                        .password(u.getPassword())
                        .roles(u.getRoles().toArray(new String[0]))
                        .build())
                .toList();
    }

    @Override
    public UserDetails loadUserByUsername(String username) {
        return users.stream()
                .filter(u -> u.getUsername().equals(username))
                .findFirst()
                .orElseThrow(() -> new UsernameNotFoundException(username));
    }
}
```

---

# ✅ Step 5：SecurityConfig（變超乾淨）

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
            .csrf().disable()
            .authorizeHttpRequests(auth -> auth
                    .anyRequest().authenticated()
            )
            .httpBasic()
            .and()
            .build();
}
```

👉 不用再手動 parse properties

# swagger 密碼問題 ⇒

1. 我不清楚為甚麼明明我有在filter裡面把swagger 的網頁設過濾premit了，代表不需要使用basic auth就可以直接進入，但我現在跑專案然後開swagger/index.html裡還是需要輸入帳號密碼

## 雷點 1：Swagger 路徑寫錯（最常見）

Swagger 不是只有 `/swagger/index.html`，它還會呼叫很多資源👇

你至少要放行這些：

```java
.requestMatchers(
    "/swagger-ui/**",
    "/v3/api-docs/**",
    "/swagger-ui.html"
).permitAll()
```

👉 很多人只寫：

```
"/swagger/index.html"
```

❌ 這是不夠的！

因為實際請求是：

- `/swagger-ui/index.html`
- `/v3/api-docs`
- `/swagger-ui/swagger-initializer.js`

---

# ✅ 正確寫法（Spring Boot 3+）

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf().disable()
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(
                "/swagger-ui/**",
                "/v3/api-docs/**",
                "/swagger-ui.html"
            ).permitAll()
            .anyRequest().authenticated()
        )
        .httpBasic();

    return http.build();
}
```

# 🧪 怎麼確認你是哪個問題？

你可以做這幾個檢查👇

---

## ✅ 檢查 1：直接打 API docs

```
http://localhost:8080/v3/api-docs
```

👉 如果這個也要登入

➡️ 代表你根本沒 permit 成功

# 💡 進階（更乾淨寫法）

如果你想寫得更漂亮：

```java
private static final String[] SWAGGER_WHITELIST = {
    "/swagger-ui/**",
    "/v3/api-docs/**",
    "/swagger-ui.html"
};

.requestMatchers(SWAGGER_WHITELIST).permitAll()
```

# metric label ⇒

1. 我目前有用helm chart yaml，然後有用普米，所以如果yaml裡面設定的labels，app: my-api，Prometheus就會帶入metric當中的label裡面的app對嗎?

> Helm yaml 設定的 labels（例如 app: my-api），
> 
> 
> 會先變成 Prometheus 的 metadata（__meta_kubernetes_pod_label_app），
> 
> 如果有透過 relabel_configs 映射，才會變成 metric label 的 app。
> 

因為目前yaml裡面所有的app的命名方式都是my-api

但我們部門對於lable的命名規則是要以底線連接，所以是my_api

所以我應該要改那些yaml的欄位才能完成這件事?

## ✅ 1️⃣ Deployment.yaml

```
metadata:
  labels:
    app: my_api# ← 要改

spec:
  selector:
    matchLabels:
      app: my_api# ← 一定要一起改（超重要）

  template:
    metadata:
      labels:
        app: my_api# ← 一定要一致
```

## ✅ 2️⃣ Service.yaml

```
spec:
  selector:
    app: my_api# ← 要改
```

# 🚨 為什麼不能只改一個地方？

因為 K8s 是靠 label match：

```
Service selector → 找 Pod label
```

如果你變成：

```
Service: app=my_api
Pod:     app=my-api
```

👉 = 完全找不到 Pod

那如果我已經用my-api這個名字建置上CICD，k8s argoCD上的話，但我現在想把app 換成my_api ，我是不是需要先把pod刪掉重建名字才能真的被更新?還是要把deployment刪掉，再重新redeploy?

## 方法 C（手動）

```
kubectl delete deployment my-api
```

然後讓 ArgoCD 自動補回來

# model

我記得我現在是用toolEventVO、toolEventDO，但其實toolEventVO裡面的欄位跟toolEventDO的欄位是完全重疊的

這支API裡面就查詢一張TABLE

像toolEventVO裡面有8個可以傳入的參數讓sql當條件過濾。DO目前則是把所有TABLE欄位OUTPUT出來

所以這邊可以改進

toolEventVO → 查詢條件（8個欄位）

toolEventDO → DB欄位（全部欄位）

改成這樣，明確定義名稱

```
ToolEventQueryDTO   ← 查詢 + validation
ToolEventResponseDTO ← 回傳
```

# ✅ 方向 1：直接用一個 DTO（最可能）

👉 把 VO / DO 合併

```
classToolEventDTO {
// 查詢條件
privateStringtype;
privateStringstatus;

// DB欄位
privateStringid;
privateStringname;
}
```

👉 一個 class 同時：

- 當 request（查詢條件）
- 當 response（查詢結果）

---

### 💡 適用情境

- 單一 table
- 欄位幾乎一樣
- 沒有複雜轉換

👉 **你現在就是這種情境**

---

# ✅ 方向 2：改名（語意清楚）

你現在叫：

```
VO（Value Object）
DO（Data Object）
```

👉 這其實在現代 Spring 專案很少這樣用

---

### 建議命名：

```
ToolEventRequest   ← 查詢條件
ToolEventResponse  ← 回傳資料
```

👉 或：

```
ToolEventQueryDTO
ToolEventDTO
```

---

👉 重點是：

👉 **名字要表達「用途」，不是技術名詞**

---

# ✅ 方向 3：Query Object + Entity（稍微進階）

👉 比較乾淨的分法：

```
classToolEventQuery {
// 只有查詢條件（8個）
}
```

```
classToolEvent {
// DB entity（完整欄位）
}
```

👉 Repository：

```
List<ToolEvent>findByQuery(ToolEventQuery query);
```

---

### 💡 好處

- 查詢條件獨立
- Entity 保持純 DB 結構
- 未來好擴展

---

# 

# 文件

圖片跟body重弄

# 欄位調整