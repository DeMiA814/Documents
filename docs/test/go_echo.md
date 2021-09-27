# GO バックエンド(echo)テストフロードキュメント

## 目次

- 単体テスト
  - 概要
  - 使用ツール
  - サンプルコード
- 結合テスト
  - 概要
  - 使用ツール
  - サンプルコード

## 単体テスト

### 概要

単体テストでのテストの単位は関数一つ一つ。echo で
実装されたもので考えると models や utils が含まれる。

### 使用ツール

1. [testing](https://pkg.go.dev/testing)
2. [testify](https://github.com/stretchr/testify)

#### testing

go 標準のテスト用のパッケージ。go のテストコマンドでテストをしたい場合はこれを使う。テストのロジック以外のさまざまな調整しなければいけないこと、例えば結果の表示などを自分で考えなくてもやってくれる。

#### testify

特に assert 機能を使う。テストのロジックを実装するときに出力に対して様々な検証をすることが想定されるが、その辺りの記述が大変楽になる。 様々な関数が用意されているのでぜひ確認してみて欲しい。[使えそうな関数](https://qiita.com/JpnLavender/items/21b4574a7513472903ea)

### サンプルコード

#### _models/user.go_

```go
package models

import (
	"time"
	"github.com/google/uuid"
)

type User struct {
	ID              uuid.UUID       `json:"id" gorm:"primary_key""`
	Name            string          `json:"name"`
	Email           string          `json:"email"`
	Icon            string          `json:"icon"`
	DateOfBirth     *time.Time      `json:"date_of_birth"`
}

func (user *User) BeforeCreate(tx *gorm.DB) (err error) {
	if user.ID == uuid.Nil {
		err = errors.New("zero value UUID is not allowed")
	}
	return
}
// Create a given user.
func (user *User) Create() error {
    return db.Create(user).Error
}

// Get a user by id.
func (user *User) Get(id uuid.UUID) error {
	return db.First(user, id).Error
}

// Update user with current values.
func (user *User) Update() error {
    return db.Save(user).Error
}

// Delete user from database.
func (user *User) Delete() error {
	return db.Delete(user).Error
}

// GetUsers returns existing users.
func GetUsers() ([]User, error) {
	var users []User
	res := db.Find(&users)
	return users, res.Error
}

// GetUserByName selects from user by name.
func GetUserByName(name string) (User, error) {
	var user User
	res := db.Where("name = ?", name).First(&user)
	return user, res.Error
}
```

このような構造体と関数を定義しているとする。

まずテスト用のファイルは対象となる関数のファイル名に"\_test"と追加した名前をつける。今回は `models/user_test.go` となる。echo では単純な CRUD は実装はほぼ作業で echo が作ってくれた部分を写すだけなので動作に関しては今回は信頼している。

#### _models/user_test.go_

```go
package models_test

import (
	"testing"
	"github.com/demia-kk/<your application name>/models"
	"github.com/stretchr/testify/assert"
)

func TestGetUsers(t *testing.T) {
	t.Run("GetUsers when users exist", func(t *testing.T) {
		names := []string{"A太郎", "B介", "C子", "D美"}
		userNum := len(names)
		if err := generateUsers(names); err != nil {
			t.Fatal(err)
		}
		users, err := models.GetUsers()
		if assert.NoError(t, err) {
			assert.Len(t, users, userNum)
		}
		if err := models.GetDBInstance().Delete(&users).Error; err != nil {
			t.Fatal(err)
		}
	})
	t.Run("GetUsers when no user exists", func(t *testing.T) {
		users, err := models.GetUsers()
		if assert.NoError(t, err) {
			assert.Len(t, users, 0)
		}
	})
}

func generateUsers(names []string) error {
	users := make([]models.User, 0, len(names))
	for _, name := range names {
		users = append(users, models.User{
			ID:   uuid.New(),
			Name: name,
		})
	}
	return models.GetDBInstance().Create(&users).Error
}
```

上記のテストコードではエラーが返ってきていないこと、ユーザー数が期待通りかのテストをしている。これを実際に`go test -v models/user_test.go`で実行すると

#### _コンソール_

```
$ go test -v ./models/user_test.go
=== RUN   TestGetUsers
=== RUN   TestGetUsers/GetUsers_when_users_exist
=== RUN   TestGetUsers/GetUsers_when_no_user_exists
--- PASS: TestGetUsers (0.00s)
    --- PASS: TestGetUsers/GetUsers_when_users_exist (0.00s)
    --- PASS: TestGetUsers/GetUsers_when_no_user_exists (0.00s)
PASS
ok      command-line-arguments  0.365s
```

のように出力される。仮にユーザーがいない時のテストで期待する数の部分を 1 に変えてみると

#### _コンソール_

```
$ go test -v ./models/user_test.go
=== RUN   TestGetUsers
=== RUN   TestGetUsers/GetUsers_when_users_exist
=== RUN   TestGetUsers/GetUsers_when_no_user_exists
    user_test.go:40:
                Error Trace:    user_test.go:40
                Error:          "[]" should have 1 item(s), but has 0
                Test:           TestGetUsers/GetUsers_when_no_user_exists
--- FAIL: TestGetUsers (0.00s)
    --- PASS: TestGetUsers/GetUsers_when_users_exist (0.00s)
    --- FAIL: TestGetUsers/GetUsers_when_no_user_exists (0.00s)
FAIL
FAIL    command-line-arguments  0.383s
FAIL
```

というふうになる。

## 結合テスト

### 概要

結合テストは複数の関数を組み合わせた結果、実装した API が期待通りに機能するかをテストする。具体的に echo では主に handlers のテストがその役割を果たす。

### 使用ツール

単体テスト時と特に変わらない。

### サンプルコード

以下のような認証周りの API が実装されているとする。

#### _handlers/auth.go_

```go
package handlers

import (
	"net/http"
	"github.com/demia-kk/<your application name>/models"
	"github.com/labstack/echo/v4"
)

// Signup is a handler for `POST /signup`.
func Signup(c echo.Context) error {
	var user models.User
	err := c.Bind(&user)
	if err != nil {
		return err
	}
	err = user.Create()
	if err != nil {
		return responseInternalServerError(c, err)
	}
	return c.JSON(http.StatusCreated, user)
}

```

単体テストの時と同様に `handlers/auth_test.go` を作成

#### _handlers/auth_test.go_

```go
package handlers

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/demia-kk/<your application name>/models"
	"github.com/google/uuid"
	"github.com/labstack/echo/v4"
	"github.com/stretchr/testify/assert"
)

func TestSignup(t *testing.T) {
	t.Run("new user with id", func(t *testing.T) {
		var user models.User
		user.ID = uuid.New()
		testSignup(t, user, http.StatusCreated, true)
	})
	t.Run("new user without id", func(t *testing.T) {
		var user models.User
		testSignup(t, user, http.StatusInternalServerError, false)
	})
}
// 同じようなテストを繰り返す時は関数に分けるのもあり
func testSignup(t *testing.T, u models.User, statusExpected int, noErrorExpected bool) {
    // このように`t.Helper()`を仕込むとエラー箇所が関数を分けて別の行になってしまってもどこで失敗したかがわかる
	t.Helper()
	uJson, err := json.Marshal(u)
	if err != nil {
		t.Fatal(err)
	}
	e := echo.New()
	req := httptest.NewRequest(http.MethodPost, "/", strings.NewReader(string(uJson)))
	req.Header.Set(echo.HeaderContentType, echo.MIMEApplicationJSON)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)
	c.SetPath("signup/")
	if noErrorExpected {
		if assert.NoError(t, Signup(c)) {
			assert.Equal(t, statusExpected, rec.Code)
			assert.JSONEq(t, string(uJson), rec.Body.String())
		}
	} else {
		if assert.Error(t, Signup(c)) {
			assert.Equal(t, statusExpected, rec.Code)
		}
	}
}

```

上記のテストコードでは正常なユーザー生成と ID がない場合のユーザー生成をテストしている。
これを実際に`go test -v handlers/auth_test.go`で実行すると

#### _コンソール_

```
$ go test -v auth.go auth_test.go utils.go errors.go
=== RUN   TestSignup
=== RUN   TestSignup/new_user_with_id
=== RUN   TestSignup/new_user_without_id

/models/user.go:47 zero value UUID is not allowed

[0.090ms] [rows:0]
--- PASS: TestSignup (0.00s)
    --- PASS: TestSignup/new_user_with_id (0.00s)
    --- PASS: TestSignup/new_user_without_id (0.00s)
PASS
ok      command-line-arguments  0.461s
```

テストのときに一時的に DB に存在するユーザーを生成したいなど、特定のセットアップが必要な状況がある。そういうときは以下のようにコードを書くこともできる。

#### _models/user_test.go_

```go
func TestHoge(t *testing.T) {
    user, teardown := setupTestHoge(t)
    defer teardown()
    // 以下テスト部分
}

func setupTestHoge(t *testing.T) (user *models.User, teardown func()) {
	user = &models.User{
        ID:   uuid.New(),
        Name: "テスト太郎",
	}
	if err := user.Create(); err != nil {
		t.Fatal(err)
	}
    teardown = func() {
		user.Delete()
	}
	return user, teardown
}

```

また、モックの DB を使いたい場合はそのためにテスト環境を整える必要がある。データの生成は models や handlers と同階層に"testdata"というディレクトリを作成する。この名前で生成されたディレクトリは通常のビルドでは無視される。

#### _階層構造_

```
testdata/
└── mock.go
models/
├── db.go
├── user.go
└── user_test.go
handlers
├── auth.go
├── auth_test.go
└── user.go
```

#### _models/db.go_

```go
package models

import (
	"bytes"
	"database/sql"
	"log"
	"time"

	"github.com/google/uuid"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
)

var db *gorm.DB

func init() {
	var err error
	db, err = gorm.Open(postgres.New(postgres.Config{
		DSN:                  "user=hoge_api password=hoge_api dbname=hoge_api port=5432 sslmode=disable TimeZone=Asia/Tokyo",
		PreferSimpleProtocol: true,
	}), &gorm.Config{
		PrepareStmt: true,
	})
	if err != nil {
		log.Fatal(err)
	}
	err = db.AutoMigrate(
		&User{},
	)
	if err != nil {
		panic("migration failed")
	}

}

func GetDBInstance() *gorm.DB {
	// 外部パッケージからプライベート変数を取得する関数
	return db
}

func SetDBInstance(newDB *gorm.DB) {
	// 外部パッケージからプライベート変数に代入する関数
	db = newDB
}

```

models で参照される db の初期化、そのインスタンスの取得、代入の実装。

#### _testdata/mock.go_

```go
package testdata

import (
	"log"
	"strconv"
	"time"

	uuid "github.com/google/uuid"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"

	"github.com/demia-kk/triceratops_api/models"
	_ "github.com/mattn/go-sqlite3"
)

var mock *gorm.DB

func Init() {
	var err error
	// Postgresの接続
	mock, err = gorm.Open(postgres.New(postgres.Config{
		DSN:                  "user=hoge_test password=hoge_test dbname=hoge_test port=5432 sslmode=disable TimeZone=Asia/Tokyo",
		PreferSimpleProtocol: true,
	}), &gorm.Config{
		PrepareStmt: true,
	})
	if err != nil {
		log.Fatal(err)
	}
	err = mock.AutoMigrate(
		&models.User{},
	)
	if err != nil {
		panic("migration failed")
	}

	models.SetDBInstance(mock) // modelsで参照されるデータベースを変更する
	generateUsers(3)   // 以下データの生成
}

func generateUsers(num int) error {
	users := make([]models.User, 0, num)
	for i := 1; i <= num; i++ {
		users = append(users, models.User{
			ID:    uuid.New(),
			Name:  "tester" + strconv.Itoa(i),
			Email: "test" + strconv.Itoa(i) + "@example.com",
		})
	}
	return mock.Create(&users).Error
}

```

mock で使用する db の初期化。 <br>
SetDBInstance で models で参照される db も mock 用に変更する。

#### _models/main_test.go_

```go
package handlers_test

import (
	"testing"

	"github.com/demia-kk/triceratops_api/testdata"
)

func TestMain(m *testing.M) {
	testdata.Init()
	m.Run()
}
```

`testdata.Init()` をテストがそれぞれ実行される前に呼ぶ必要がある。このときに便利なのが `*testing.M` 。go のテストでは`TestMain`が存在するときそれがまず初めに実行される。そして、`testing.M`から`Run()`が呼ばれると各ファイルのテストコードが実行される。
