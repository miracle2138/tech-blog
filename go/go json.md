```
import (
	"encoding/json"
	"fmt"
	"testing"
)

// 可以使用struct tag指定json序列化/反序列化的特性
// tag是go语言的特性，可以在反射时拿到tag信息，类似java的注解

type LiYaoPerson struct {
	Name    string   `json:"myName"`            // 指定别名
	Region  string   `json:"-"`                 // 忽略字段
	Address []string `json:"address,omitempty"` // 忽略空值，空值也即零值；注意这里必须指定别名
}

func TestJsonMarshal(t *testing.T) {
	p := LiYaoPerson{
		Name:   "haha",
		Region: "cn",
	}

	marshal, err := json.Marshal(p)
	if err != nil {
		return
	}

	s := string(marshal)
	fmt.Println(s)
}

func TestJsonUnMarshal(t *testing.T) {
	s := "{\"myName\":\"haha\"}"

	var p = LiYaoPerson{}
	_ = json.Unmarshal([]byte(s), &p)

	fmt.Printf("model is %+v\n", p)
}
```
