建立 Map 時： 用全 Stream 寫法，合併物件時： 用建構子（Constructor）
```
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class JoinMethodCombined {
    public static void main(String[] args) {
        // 模擬資料
        List<Equipment> eqpList = Arrays.asList(
            new Equipment("E001", "機台A"),
            new Equipment("E002", "機台B"),
            new Equipment("E003", "機台C")
        );

        List<Status> statusList = Arrays.asList(
            new Status("E001", "Running"),
            new Status("E002", "Idle"),
            new Status("E004", "Down")
        );

        // 1. 全 Stream 建立 Map
        // 把 statusList 變成 Map
        Map<String, Status> statusMap = statusList.stream()
            .collect(Collectors.toMap(
                Status::getEqpId, // 指定key是哪一個
                Function.identity(), // 整個 Status 物件都放進去
                (oldVal, newVal) -> newVal // 安全機制：重複時保留最新資料，感覺可以先拿掉，有問題再加
            ));

        // 2. 處理與合併 (利用建構子一行精簡化)
        List<CombinedData> result = eqpList.stream()
            .filter(eqp -> statusMap.containsKey(eqp.getEqpId())) // Inner Join 篩選，只保留在 statusMap 找得到的 eqp，拿掉就是left join
            .map(eqp -> new CombinedData(eqp, statusMap.get(eqp.getEqpId()))) // 一行直接 new 出來
            .collect(Collectors.toList());
    }
}
```
接下來給我 串起來需要兩個欄位合併的版本 inner join 的範例
```
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class DoubleFieldsJoin {
    public static void main(String[] args) {
        // 模擬 List A
        List<Equipment> eqpList = Arrays.asList(
            new Equipment("E001", "F1", "機台A-一廠"),
            new Equipment("E001", "F2", "機台A-二廠"), // 同 ID，不同廠
            new Equipment("E002", "F1", "機台B-一廠")
        );

        // 模擬 List B
        List<Status> statusList = Arrays.asList(
            new Status("E001", "F1", "Running"), // 對得到 (E001 + F1)
            new Status("E001", "F3", "Idle"),    // 對不到 (雖然 ID 是 E001 但廠區是 F3)
            new Status("E002", "F1", "Down")     // 對得到 (E002 + F1)
        );

        // 1. 全 Stream 建立 Map：將兩個欄位用底線 "_" 拼起來當作 Key
        Map<String, Status> statusMap = statusList.stream()
            .collect(Collectors.toMap(
                status -> status.getEqpId() + "_" + status.getFactoryCode(), // 複合 Key
                Function.identity(),
                (oldVal, newVal) -> newVal // 重複時覆蓋
            ));

        // 2. 處理與合併：比對時用同樣的規則拼出 Key 去查詢
        List<CombinedData> result = eqpList.stream()
            .filter(eqp -> statusMap.containsKey(eqp.getEqpId() + "_" + eqp.getFactoryCode())) // 雙欄位比對
            .map(eqp -> new CombinedData(eqp, statusMap.get(eqp.getEqpId() + "_" + eqp.getFactoryCode()))) // 建構子合併
            .collect(Collectors.toList());

        // 印出結果 (只會成功合併 2 筆)
        result.forEach(System.out::println);
    }

```


接下來我希望通過比對 狀態為 Running、Down的，IsKey 設Y，其他設N
```
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class DoubleFieldsLeftJoinWithLogic {
    public static void main(String[] args) {
        // 模擬 List A
        List<Equipment> eqpList = Arrays.asList(
            new Equipment("E001", "F1", "機台A-一廠"),
            new Equipment("E001", "F2", "機台A-二廠"), // 沒對到狀態
            new Equipment("E002", "F1", "機台B-一廠"),
            new Equipment("E003", "F1", "機台C-一廠")  // 狀態為 Idle
        );

        // 模擬 List B
        List<Status> statusList = Arrays.asList(
            new Status("E001", "F1", "Running"), // 符合 Running -> Y
            new Status("E002", "F1", "Down"),    // 符合 Down -> Y
            new Status("E003", "F1", "Idle")     // 其他狀態 -> N
        );

        // 1. 全 Stream 建立 Map (雙欄位複合 Key)
        Map<String, Status> statusMap = statusList.stream()
            .collect(Collectors.toMap(
                status -> status.getEqpId() + "_" + status.getFactoryCode(),
                Function.identity(),
                (oldVal, newVal) -> newVal
            ));

        // 2. 處理、Left Join、並根據狀態判斷 IsKey
        List<CombinedData> result = eqpList.stream()
            .map(eqp -> {
                CombinedData joined = new CombinedData();
                joined.setEqpId(eqp.getEqpId());
                joined.setFactoryCode(eqp.getFactoryCode());
                joined.setName(eqp.getName());

                // 取得對應的狀態
                String key = eqp.getEqpId() + "_" + eqp.getFactoryCode();
                Status status = statusMap.get(key);

                if (status != null) {
                    String currentState = status.getState();
                    joined.setState(currentState);

                    // ★ 核心邏輯：比對狀態為 Running 或 Down 則設 Y，其他設 N
                    if ("Running".equalsIgnoreCase(currentState) || "Down".equalsIgnoreCase(currentState)) {
                        joined.setIsKey("Y");
                    } else {
                        joined.setIsKey("N");
                    }
                } else {
                    // Left Join 沒對到資料的情況（status 為 null）
                    joined.setState("Unknown");
                    joined.setIsKey("N"); // 沒狀態當然也不是 Running/Down，設 N
                }

                return joined;
            })
            .collect(Collectors.toList());
    }
}
```
