# 負荷ツール vegeta(ベジータ)の使い方

## 目次

- 負荷テスト
  - 概要
  - 使用ツール
  - 手順
  - サンプルコード
- echo のとき
- django のとき

## 負荷テスト

### 概要

このテストでは想定される負荷をバックエンドにかけてそのパフォーマンスを確認する。同時リクエストなど非同期的なテストが必要になってくる。今回紹介する方法で実装すればほぼコピペで動くがお決まりの設計みたいな感じのものがなく自前設計なのでより良い案があれば随時募集。

### 使用ツール

1. [vegeta](https://github.com/tsenart/vegeta)

<br>
特定のエンドポイントに対してリクエストのペースと時間を指定して攻撃することができる。その結果が返ってくるが、その中には実際に実行した攻撃のログ、待ち時間、ステータスコードなどの情報が含まれている。

### 手順

1. 必要な情報を用意する

- 攻撃する URL
- リクエストの内容
- リクエストの頻度

2. 必要なディレクトリ、ファイルの作成

```
myapp/attacktests
├── attacks
│   ├── attack.go
│   ├── base.go
│   ├── constants.go
│   ├── season.go(今回は例)
│   └── signup.go(今回は例)
└── main.go
```

3. テスト用に用意された`MyAttack`を委譲した構造体で`Attack()`というメソッドを定義し、その中に具体的な攻撃内容を記述する。

### サンプルコード

`MyAttack`は攻撃に必要な情報を持つ構造体。

#### _attacks/attack.go_

```go
package attacks

import (
    // 省略
)

type MyAttackInterface interface {
	Attack()
}

type MyAttack struct {
	Rate     vegeta.Pacer
	Duration time.Duration
	Method   string
	URL      string
	Body     []byte
	Header   http.Header
}

type MyAttackWithAuth struct {
	MyAttack
	AuthorizedTesters
}

type MyAttackWitSingletonAuth struct {
	MyAttack
	SingletonAuthorizedTesters
}

```

各テストにこの`MyAttack`が委譲されるがそれぞれの`Attack()`から以下の`doAttack()`を呼び出す。

#### _attacks/attack.go_

```go
func doAttack(targeter vegeta.Targeter, rate vegeta.Pacer, dur time.Duration, name string) {
	var (
		metrics vegeta.Metrics
		wg      sync.WaitGroup
		mtx     sync.Mutex
	)
	defer metrics.Close()
	wg.Add(1)
	go func() {
		defer wg.Done()
		for result := range vegeta.NewAttacker().Attack(targeter, rate, dur, name) {
			if result == nil || result.Error == vegeta.ErrNoTargets.Error() {
				continue
			}
			mtx.Lock()
			metrics.Add(result)
			mtx.Unlock()
		}
	}()
	wg.Wait()
	b, err := json.Marshal(metrics)
	if err != nil {
		panic(err)
	}
	println("\n========")
	println(name)
	os.Stdout.Write(b)
	println("\n========")
}
```

具体的なテストの作り方は

#### _attacks/signup.go_

```go
package attacks

import (
    // 省略
)

type SignupAttacks struct {
    // signup関連のテストをまとめるための構造体
    // テストが追加されればその分フィールドを追加する。
    // `MyAttackInterface`型なのは`Attack()`があることを確かにするため
	UserCreationAttack　MyAttackInterface
}

func NewSignupAttacks() *SignupAttacks {
	return &SignupAttacks{
		UserCreationAttack: newUserCreationAttack(),
	}
}

type UserCreationAttack struct {
	MyAttack
	newUsers []*models.User
}

func newUserCreationAttack() *UserCreationAttack {
	a := new(UserCreationAttack)
	a.Rate = vegeta.Rate{Freq: 100, Per: time.Second}
	a.Duration = time.Duration(1) * time.Second
	a.Method = http.MethodPost
	a.Header = http.Header{
		"Content-Type": {"application/json"},
	}
	a.URL = signupURL
	a.newUsers = generateNewUsers(100)
	return a
}

```

`UserCreationAttack`が実際にテストを作っていく部分で`MyAttack`を埋め込んでいる。それに加えてテスト固有のフィールドを定義している。<br>
この構造体に対して`Attack()`を定義して`MyAttackInterface`を実装していくことになる。

#### _attacks/signup.go_

```go
func (a *UserCreationAttack) Attack() {
	doAttack(a.generateTargeter(), a.Rate, a.Duration, fmt.Sprint(reflect.TypeOf(a)))
}

func (a *UserCreationAttack) generateTargeter() vegeta.Targeter {
	var targets []*vegeta.Target
	for _, u := range a.newUsers {
		if u == nil {
			panic("user is null.")
		}
		userJson, err := json.Marshal(u)
		if err != nil {
			panic(err)
		}
		target := &vegeta.Target{
			Method: a.Method,
			Header: a.Header,
			URL:    a.URL,
			Body:   userJson,
		}
		targets = append(targets, target)

	}
	return targeterFromTargets(targets)
}
```

#### _attacks/attack.go_

```go
func targeterFromTargets(targets []*vegeta.Target) vegeta.Targeter {
	var (
		mu    sync.Mutex
		index int
	)

	return func(tgt *vegeta.Target) error {
		mu.Lock()
		defer mu.Unlock()
		defer func() {
			index++
		}()

		if index >= len(targets) {
			return vegeta.ErrNoTargets
		}
		t := targets[index]
		tgt.Method, tgt.URL, tgt.Header = t.Method, t.URL, t.Header
		if t.Body != nil {
			tgt.Body = t.Body
		}
		return nil
	}
}
```

`vegeta.Targeter`は`func(tgt *vegeta.Target) error`である関数が満たす型。多くの場合はリクエストの情報を持った target はテストにつきひとつでそれを攻撃に用いるが、今回のサインアップのようにリクエスト時点で id を決めなければならずユーザー分のリクエストの種類が必要なときのために`targeterFromTargets`を実装した。基本的に`generateTargeter`以下のようになる。

```go
func (a *GetSeassonsAttack) generateTargeter() vegeta.Targeter {
	a.Header.Add("Authorization", a.Tester1.AuthToken)
    // targetは1つ
	t := &vegeta.Target{
		Method: a.Method,
		Header: a.Header,
		URL:    a.URL,
	}
	return func(tgt *vegeta.Target) error {
		tgt.Method, tgt.URL, tgt.Header = t.Method, t.URL, t.Header
		return nil
	}
}
```

大事なのは指定した情報を含む`*vegeta.Target`を引数に持ち、error を返す関数を返すこと。<br>
認証情報が必要な場合は以下のような実装をする必要がある。

#### _attacks/attack.go_

```go
type MyAttackWithAuth struct {
    // 構造体のインスタンスを生成する度にユーザーは変わる
	MyAttack
	AuthorizedTesters
}

type MyAttackWithSingletonAuth struct {
    // 使うユーザーはずっと同じ
	MyAttack
	SingletonAuthorizedTesters
}

```

これらのいずれかをテストの構造体に埋め込むことで構造体変数としてログイン中のユーザーを使用することができる。

#### _attacks/base.go_

```go
package attacks

import (
    // 省略
)

type Tester struct {
	models.User
	AuthToken string
}

type loginResponse struct {
	Token string `json:"token"`
}

func (t *Tester) login() error {
	var (
		client http.Client
		res    *http.Response
		lr     loginResponse
	)
	userJson, err := json.Marshal(t.User)
	if err != nil {
		return err
	}
	req, err := http.NewRequest(http.MethodPost, loginURL, bytes.NewReader(userJson))
	req.Header.Set("Content-Type", "application/json")
	if err != nil {
		return err
	}
	res, err = client.Do(req)
	if err != nil {
		return err
	}
	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return err
	}
	err = json.Unmarshal(body, &lr)
	if err != nil {
		return err
	}
	t.AuthToken = fmt.Sprintf("Bearer %s", lr.Token)
	return nil
}
```

`Tester`は`User`を埋め込んでいる。認証情報のトークンを保持するための AuthToken フィールドをもつ。<br>

#### _attacks/base.go_

```go
type AuthorizedTesters struct {
    // ユーザーの数は暫定的に2にしたが必要に応じて調整する
	Tester1 *Tester
	Tester2 *Tester
}

func NewAuthorizedTesters() AuthorizedTesters {
    // 初期化
	ts := new(AuthorizedTesters)
	if err := ts.setup(); err != nil {
		panic(err)
	}
	return *ts
}


func (b *AuthorizedTesters) setup() error {
	b.Tester1 = &Tester{
		User: models.User{
			ID:    uuid.New(),
			Name:  "tester1",
			Email: "tester1@tester1.com",
		},
	}
	if err := b.Tester1.Create(); err != nil {
		return err
	}
	if err := b.Tester1.login(); err != nil {
		return err
	}
	b.Tester2 = &Tester{
		User: models.User{
			ID:    uuid.New(),
			Name:  "tester2",
			Email: "tester2@tester2.com",
		},
	}
	if err := b.Tester2.Create(); err != nil {
		return err
	}
	if err := b.Tester2.login(); err != nil {
		return err
	}
	return nil
}

```

#### _attacks/base.go_

```go

type SingletonAuthorizedTesters struct {
	AuthorizedTesters
}

var instance *SingletonAuthorizedTesters

func GetSingletonAuthorizedTesters() SingletonAuthorizedTesters {
    // この関数を通すことで常に同じインスタンスを返すようにしている
	if instance == nil {
		instance = initSingletonAuthorizedTesters()
	}
	return *instance
}

func initSingletonAuthorizedTesters() *SingletonAuthorizedTesters {
	sts := new(SingletonAuthorizedTesters)
	sts.AuthorizedTesters = NewAuthorizedTesters()
	return sts
}

```

実際に認証情報が必要なテストを実装してみると

#### _attacks/season.go_

```go
package attacks

import (
    // 省略
)

type SeasonsAttacks struct {
	GetSeassonsAttack MyAttackInterface
}

func NewSeasonsAttacks() *SeasonsAttacks {
	as := new(SeasonsAttacks)
	as.GetSeassonsAttack = NewGetSeassonsAttack()
	return as
}

type GetSeassonsAttack struct {
    // 埋め込み
	MyAttackWithAuth
}

func NewGetSeassonsAttack() *GetSeassonsAttack {
	a := new(GetSeassonsAttack)
    // ここで認証されたユーザーの初期化を行う
	a.AuthorizedTesters = NewAuthorizedTesters()
	a.Rate = vegeta.Rate{Freq: 10, Per: time.Second}
	a.Duration = time.Duration(1) * time.Second
	a.Method = http.MethodGet
	a.Header = http.Header{
		"Content-Type": {"application/json"},
	}
	a.URL = seasonsURL
	return a
}

func (a *GetSeassonsAttack) Attack() {
	doAttack(a.generateTargeter(), a.Rate, a.Duration, fmt.Sprint(reflect.TypeOf(a)))
}

func (a *GetSeassonsAttack) generateTargeter() vegeta.Targeter {
	a.Header.Add("Authorization", a.Tester1.AuthToken)
	t := &vegeta.Target{
		Method: a.Method,
		Header: a.Header,
		URL:    a.URL + "?q=1&o=999&t_q=&r_s=0&r_e=5",
	}
	return func(tgt *vegeta.Target) error {
		tgt.Method, tgt.URL, tgt.Header = t.Method, t.URL, t.Header
		return nil
	}
}
```

呼び出し部分は

#### _attacks/main.go_

```go
package main

import (
	"<module name>/attacks"
)

func main() {
	signupAttacks := attacks.NewSignupAttacks()
	signupAttacks.UserCreationAttack.Attack()
	seasonsAttacks := attacks.NewSeasonsAttacks()
	seasonsAttacks.GetSeassonsAttack.Attack()
}
```

これらを最終的に実行すると(サーバーを立てておくのを忘れない)

```
$ go run ./attacktests
```

```
========
*attacks.UserCreationAttack
{"latencies":{"total":79721833,"mean":0,"50th":0,"95th":0,"99th":0,"max":23259687},"bytes_in":{"total":2760,"mean":0},"bytes_out":{"total":2750,"mean":0},"earliest":"2021-09-16T05:16:19.869059637+09:00","latest":"2021-09-16T05:16:20.768944903+09:00","end":"2021-09-16T05:16:20.771548828+09:00","duration":0,"wait":0,"requests":10,"rate":0,"throughput":0,"success":0,"status_codes":{"201":10},"errors":[]}
========

========
*attacks.GetSeassonsAttack
{"latencies":{"total":32090318,"mean":0,"50th":0,"95th":0,"99th":0,"max":5004221},"bytes_in":{"total":30,"mean":0},"bytes_out":{"total":0,"mean":0},"earliest":"2021-09-16T05:16:20.896618132+09:00","latest":"2021-09-16T05:16:21.792598761+09:00","end":"2021-09-16T05:16:21.794190783+09:00","duration":0,"wait":0,"requests":10,"rate":0,"throughput":0,"success":0,"status_codes":{"200":10},"errors":[]}
========
```

このように攻撃結果が返ってくる。レイテンシーが気になるところで単位はナノ秒。

## echo のとき

サンプルコード通りで問題ない。

## django のとき

ログイン不要部分の GET などの実装はあまり変わらないがログイン必須の部分や POST の部分などで上記のコードでは対応できないので Djnago に合わせた実装に変更する必要がある。

(例)

```
myproject/attacktests
├── attacks
│   ├── attack.go
│   ├── base.go
│   ├── blog.go
│   ├── login.go
│   ├── signup.go
│   ├── urls.go
│   └── utils.go
├── go.mod
├── go.sum
└── main.go
```

ブログ一覧をログイン必須の`/top`で確認できる簡単な django のプロジェクトを想定。

#### _attacks/base.go_

```go
package attacks

import (
	// 省略
)

type Tester struct {
	User          url.Values
	SessionCookie *http.Cookie
}
```

go で実装していないアプリケーションではモデルの構造体をそのまま使うことはできないが基本的にはリクエストのデータを揃えれば良いケースばかりなので`url.Values`で大体は代用できる。

#### _attacks/base.go_

```go

func (t *Tester) login() error {
	client := &http.Client{
		// リダイレクトしないようにする
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
	csrfCookie, err := getCSRFCookie(loginURL)
	if err != nil {
		return err
	}
	req, err := http.NewRequest(http.MethodPost, loginURL, bytes.NewReader([]byte(t.User.Encode())))
	if err != nil {
		return err
	}
	req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
	setCSRFToCookieAndHeader(req, *csrfCookie)
	res, err := client.Do(req)
	if err != nil {
		return err
	}
	sessionCookie, err := getSessionCookie(res)
	if err != nil {
		return err
	}
	t.SessionCookie = sessionCookie
	return nil
}

func (t *Tester) signup() error {
	var (
		client http.Client
		req    *http.Request
	)
	csrfCookie, err := getCSRFCookie(signupURL)
	if err != nil {
		log.Fatal(err)
	}
	req, _ = http.NewRequest(http.MethodPost, signupURL, bytes.NewReader([]byte(t.User.Encode())))
	req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
	req.AddCookie(csrfCookie)
	req.Header.Set("X-CSRFToken", csrfCookie.Value)
	_, err = client.Do(req)
	return err
}

type AuthorizedTesters struct {
	Tester1 *Tester
	Tester2 *Tester
}

func (ts *AuthorizedTesters) setup() error {
	ts.Tester1 = &Tester{
		User: url.Values{
			"username":  []string{"t太郎"},
			"email":     []string{"tester1@tester.com"},
			"password":  []string{"thisistest"},
			"password1": []string{"thisistest"},
			"password2": []string{"thisistest"},
		}}
	if err := ts.Tester1.signup(); err != nil {
		return err
	}
	if err := ts.Tester1.login(); err != nil {
		return err
	}
	return nil
}

func NewAuthorizedTesters() AuthorizedTesters {
	ts := new(AuthorizedTesters)
	if err := ts.setup(); err != nil {
		panic(err)
	}
	return *ts
}

type SingletonAuthorizedTesters struct {
	AuthorizedTesters
}

var instance *SingletonAuthorizedTesters

func GetSingletonAuthorizedTesters() SingletonAuthorizedTesters {
	if instance == nil {
		instance = initSingletonAuthorizedTesters()
	}
	return *instance
}

func initSingletonAuthorizedTesters() *SingletonAuthorizedTesters {
	sts := new(SingletonAuthorizedTesters)
	sts.AuthorizedTesters = NewAuthorizedTesters()
	return sts
}

```

django は csrf 対策がされていてログイン状態も`sessionid`という Cookie で管理されているのでその部分の実装を変更する必要があった。

#### _attacks/utils.go_

```go
package attacks

import (
	// 省略
)

func getCSRFCookie(url string) (*http.Cookie, error) {
	res, err := http.Get(url)
	if err != nil {
		log.Fatal(err)
	}
	for _, cookie := range res.Cookies() {
		if cookie.Name == "csrftoken" {
			return cookie, nil
		}
	}
	return nil, errors.New("no CSRF Cookie")
}

func getSessionCookie(res *http.Response) (*http.Cookie, error) {
	for _, cookie := range res.Cookies() {
		if cookie.Name == "sessionid" {
			return cookie, nil
		}
	}
	return nil, errors.New("no sessionid Cookie")
}

func setCSRFToCookieAndHeader(req *http.Request, c http.Cookie) {
	req.AddCookie(&c)
	req.Header.Set("X-CSRFToken", c.Value)
}

```

#### _attacks/signup.go_

```go
package attacks

import (
	// 省略
)

type SignupAttacks struct {
	UserCreationAttack MyAttackInterface
}

func NewSignupAttacks() *SignupAttacks {
	return &SignupAttacks{
		UserCreationAttack: newUserCreationAttack(),
	}
}

type UserCreationAttack struct {
	MyAttack
	newUsers []url.Values
}

func newUserCreationAttack() *UserCreationAttack {
	a := new(UserCreationAttack)
	a.Rate = vegeta.Rate{Freq: 100, Per: time.Second}
	a.Duration = time.Duration(1) * time.Second
	a.Method = http.MethodPost
	a.Header = http.Header{
		"Content-Type": {"application/json"},
	}
	a.URL = signupURL
	a.newUsers = generateNewUsers(100)
	return a
}

func (a *UserCreationAttack) Attack() {
	doAttack(a.generateTargeter(), a.Rate, a.Duration, fmt.Sprint(reflect.TypeOf(a)))
}

func (a *UserCreationAttack) generateTargeter() vegeta.Targeter {
	var targets []*vegeta.Target
	csrfCookie, err := getCSRFCookie(signupURL)
	if err != nil {
		log.Fatal(err)
	}
	a.Header.Set("Content-Type", "application/x-www-form-urlencoded")
	a.Header.Set("X-CSRFToken", csrfCookie.Value)
	a.Header.Set("Cookie", fmt.Sprint(csrfCookie))
	for _, u := range a.newUsers {
		if u == nil {
			panic("user is null.")
		}
		if err != nil {
			log.Fatal(err)
		}
		target := &vegeta.Target{
			Method: a.Method,
			Header: a.Header,
			URL:    a.URL,
			Body:   []byte(u.Encode()),
		}
		targets = append(targets, target)

	}
	return targeterFromTargets(targets)
}

func generateNewUsers(userNum int) (newUsers []url.Values) {
	for i := 0; i < userNum; i++ {
		newUsers = append(newUsers, url.Values{
			"username":  []string{faker.Name() + "太郎"},
			"email":     []string{faker.Email()},
			"password1": []string{"thisistest"},
			"password2": []string{"thisistest"},
		})
	}
	return newUsers
}

```

User として扱うものが`url.Values`になっている。

#### _attacks/blog.go_

```go

package attacks

import (
	// 省略
)

type BlogAttacks struct {
	BlogListAttack MyAttackInterface
}

func NewBlogAtatcks() *BlogAttacks {
	return &BlogAttacks{
		BlogListAttack: newBlogListAttack(),
	}
}

type BlogListAttack struct {
	MyAttackWithSingletonAuth
}

func newBlogListAttack() *BlogListAttack {
	a := new(BlogListAttack)
	a.SingletonAuthorizedTesters = GetSingletonAuthorizedTesters()
	a.Rate = vegeta.Rate{Freq: 100, Per: time.Second}
	a.Duration = time.Duration(1) * time.Second
	a.Method = http.MethodGet
	a.Header = http.Header{}
	a.URL = topURL
	return a
}

func (a *BlogListAttack) Attack() {
	doAttack(a.generateTargeter(), a.Rate, a.Duration, fmt.Sprint(reflect.TypeOf(a)))
}

func (a *BlogListAttack) generateTargeter() vegeta.Targeter {
	// ここでログイン状態を作っている。
	a.Header.Set("Cookie", a.Tester1.SessionCookie.Raw)
	t := &vegeta.Target{
		Method: a.Method,
		Header: a.Header,
		URL:    a.URL,
	}
	return func(tgt *vegeta.Target) error {
		tgt.Method, tgt.URL, tgt.Header = t.Method, t.URL, t.Header
		return nil
	}
}
```

`a.Header.Set("Cookie", a.Tester1.SessionCookie.Raw)`の部分で`Tester1`がログインしている状態でリクエストを送れるようにしている。
