BCryptPasswordEncoder 實現使用廣泛支援的 bcrypt 演算法對密碼進行雜湊處理。為了使其更能抵抗密碼破解，bcrypt 被故意設計為緩慢的。像其他自適應單向函式一樣，它應該調整為在您的系統上驗證密碼大約需要 1 秒。BCryptPasswordEncoder 的預設實現使用強度 10，如 BCryptPasswordEncoder 的 Javadoc 中所述。建議您在自己的系統上調整和測試強度引數，以便驗證密碼大約需要 1 秒。
________________________________________
________________________________________
3️⃣ BCrypt 是怎麼 encode 的？
BCrypt 的特性：
1.	會自動加 salt
2.	每次 encode 結果都不同
3.	不可逆（hash）
________________________________________
encode 流程（簡化版）
原始密碼: 123456

1️⃣ 產生 random salt
2️⃣ cost factor（預設 10）
3️⃣ 做多輪 hash（Blowfish-based）
4️⃣ 輸出：

$2a$10$[salt][hash]
加上 Spring Security 前綴：
{bcrypt}$2a$10$abcxyz....
________________________________________
解析加密字串
https://ithelp.ithome.com.tw/articles/10196477
 
加密後的 bcrypt 分為四個部分：
•	Bcrypt 
o	該字串為 UTF-8 編碼，並且包含一個終止符
•	Round 
o	(回合數)每增加一次就加倍雜湊次數，預設10次
•	Salt 
o	(加鹽)128 bits 22個字元
•	Hash 
o	(雜湊)138 bits 31個字元
🧠 一、系統啟動（建立使用者）
你寫：
User.withDefaultPasswordEncoder()
.username("user")
.password("123456")
.build();
實際發生：
👉 呼叫
BCryptPasswordEncoder
________________________________________
🔐 bcrypt 做的事
1️⃣ 產生一個 隨機 salt（重點）
2️⃣ 設定 cost（預設 10）
3️⃣ 用密碼 + salt 去做 hash
________________________________________
📦 最後產生一串字：
{bcrypt}$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
________________________________________
🔍 這串東西其實有結構（超重要🔥）
$2a$10$N9qo8uLOickgx2ZMRZoMye | IjZAgcfl7p92ldGxad68LJZdL17lhWy
        ↑ salt              ↑ hash
👉 bcrypt 把：
•	salt
•	cost
•	hash
👉 全部包在同一個字串裡
🧠 二、登入時發生什麼事？
當你呼叫：
passwordEncoder.matches("123456",storedPassword)
例如：
matches("123456", "{bcrypt}$2a$10$N9qo8uLOickg...")
________________________________________
⚙️ 內部流程（完整）
1️⃣ 解析 {bcrypt}
👉 找到對應 encoder（DelegatingPasswordEncoder）
________________________________________
2️⃣ 把 bcrypt 字串拆開
$2a$10$N9qo8uLOickgx2ZMRZoMye...
        ↑ 這裡就是 salt
👉 bcrypt 自己會：
•	解析 cost（10）
•	抓出 salt
________________________________________
3️⃣ 用「同一個 salt」重新 hash
hash( "123456" + 剛剛取出的 salt )
👉 這一步最花時間（你前面問的 50~100ms）
👉 cost = bcrypt 的「運算強度（work factor）」
👉 數字越大：
•	✔ 越安全 
•	❌ 越慢 
________________________________________
🔍 cost = 10 到底在幹嘛？
bcrypt 的設計是：
運算次數 ≈ 2^cost
所以：
•	cost = 10 → 約 2¹⁰ = 1024 次運算 
•	cost = 11 → 約 2048 次 
•	cost = 12 → 約 4096 次 

________________________________________
4️⃣ 比對結果
新算出來的 hash === 原本的 hash ?
✔ 一樣 → 登入成功
❌ 不一樣 → 失敗
________________________________________
🎯 關鍵結論（你卡住的點）
👉 系統沒有另外存 salt
👉 👉 salt 就在 hash 字串裡
4️⃣ 驗證密碼怎麼做？
當你 login：
passwordEncoder.matches(rawPassword,encodedPassword)
例如：
matches("123456","{bcrypt}$2a$10$abc...")
流程是：
1.	讀 {bcrypt} → 找對應 encoder
2.	拿出 salt（存在 hash 裡）
3.	用同樣方式 hash 一次
4.	比對結果
________________________________________
5️⃣ 為什麼官方說它不安全？
因為：
❌ 問題 1：寫法會讓人誤用
.withDefaultPasswordEncoder()
👉 很多人以為這是「推薦做法」
________________________________________
❌ 問題 2：密碼通常寫死在 code 裡
.password("123456")
👉 超危險（明文）
________________________________________
❌ 問題 3：不可控
你不能：
•	調整 cost factor
•	自訂 encoder
•	管理升級策略
________________________________________
6️⃣ 官方建議怎麼做？
👉 自己定義 PasswordEncoder
	@Bean
	publicPasswordEncoderpasswordEncoder() {
	returnnewBCryptPasswordEncoder();
	}
然後：
User.builder()
.username("user")
.password(passwordEncoder().encode("123456"))
.roles("USER")
.build();
________________________________________
7️⃣ InMemoryUserDetailsManager 跟它的關係
InMemoryUserDetailsManager
它只是：
👉 把 UserDetails 存在記憶體裡
完全不負責 encode
👉 encode 是在你 .password() 時就完成了
________________________________________
🎯 一句話總結
withDefaultPasswordEncoder() =
👉 幫你用 DelegatingPasswordEncoder（預設 bcrypt）自動 encode 密碼 + 加上 {bcrypt} 前綴
但：
👉 只是為了 demo，實務上請自己控制 PasswordEncoder
Parsing

