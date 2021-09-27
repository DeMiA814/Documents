# Flutter テストフロードキュメント

## 目次

- はじめに
- 単体テスト
  - 概要
  - 使用ツール
  - 手順
  - サンプルコード
- Widget テスト
  - 概要
  - 使用ツール
  - 手順
  - サンプルコード
- 結合テスト
  - 概要
  - 使用ツール
  - 手順
  - サンプルコード

## はじめに

```yaml
dev_dependencies:
  test:
  flutter_test:
    sdk: flutter
  flutter_driver:
    sdk: flutter
```

test パッケージは最新だと他のツールとうまくいかなくなるので現状はその部分を pub が調整してくれる書き方にしている。動作確認できる最新版が出たら変更する可能性あり。

テスト用の依存パッケージをインストール。

今回のサンプルの階層構造は

```
lib
├── controllers
│   └── counter.dart
├── main.dart
└── pages
    └── home.dart
test
├── unit_test
│   └── controllers
│       └── counter_test.dart
└── widget_test
    └── pages
        └── home_test.dart
test_driver
├── app.dart
└── app_test.dart
```

#### _lib/main.dart_

```dart
import 'package:flutter/material.dart';
import 'package:test_app/pages/home.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: HomePage(),
    );
  }
}

```

flutter の最初に生成されるアプリを少し変更したものを作成した。
主な UI の部分の記述は以下の`HomePage`クラスにある。DeMiA での実装を考慮した。

#### _lib/pages/home.dart_

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:test_app/controllers/counter.dart';

class HomePage extends StatelessWidget {
  HomePage({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => CounterController(),
      builder: (context, _) {
        return Scaffold(
          appBar: AppBar(
            title: const Text('Counter App'),
          ),
          body: Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                const Text('You have pushed the button this many times:'),
                Text(
                  '${context.watch<CounterController>().value}',
                  style: Theme.of(context).textTheme.headline4,
                ),
              ],
            ),
          ),
          floatingActionButton: SizedBox(
            height: 200,
            child: Column(
              children: [
                Expanded(
                  child: FloatingActionButton(
                    key: const ValueKey('increment'),
                    onPressed: context.read<CounterController>().increment,
                    tooltip: 'Increment',
                    child: const Icon(Icons.add),
                  ),
                ),
                if (context.read<CounterController>().isPositive)
					//減らせる時だけボタンは出てくる。
                  Expanded(
                    child: FloatingActionButton(
                      key: const ValueKey('decrement'),
                      onPressed: context.read<CounterController>().decrement,
                      tooltip: 'Decrement',
                      child: const Icon(Icons.remove),
                    ),
                  ),
              ],
            ),
          ),
        );
      },
    );
  }
}

```

## 単体テスト

### 概要

flutter 開発での単体テストでは

1. クラス外で定義された関数
2. メソッド

のテストを行う

### 使用ツール

1. [flutter_test](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html)

### 手順

1. テストファイルの作成

2. テストの記述
   - `expect()`での評価

### サンプルコード

`CounterController`クラスのメソッドについてのテストコードを書いていく。

#### _lib/controllers/counter.dart_

```dart
import 'package:flutter/material.dart';

class CounterController extends ChangeNotifier {
  int _value = 0;
  int get value => _value;
  set value(int v) {
    _value = v;
    notifyListeners();
  }

  bool get isPositive => _value > 0;

  void increment() {
    _value++;
    notifyListeners();
  }

  void decrement() {
    if (_value == 0) {
      return;
    }
    _value--;
    notifyListeners();
  }
}
```

#### _test/unit_test/controllers/counter_test.dart_

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:test_app/controllers/counter.dart';

void main() {
  group('CounterControllerTests', () {
    CounterControllerTests.testIncrement();
    CounterControllerTests.testDecrement();
    CounterControllerTests.testLessThanZero();
  });
}

class CounterControllerTests {
  static void testIncrement() {
    test('counter increment test', () {
      final controller = CounterController();
      controller.increment();
      expect(controller.value, 1);
    });
  }

  static void testDecrement() {
    test('counter decrement test', () {
      final controller = CounterController();
      controller.value = 1;
      controller.decrement();
      expect(controller.value, 0);
    });
  }

  static void testLessThanZero() {
    test('counter increment test', () {
      final controller = CounterController();
      controller.decrement();
      expect(controller.value, 0);
    });
  }
}
```

`〇〇_test.dartの`というファイルの`main()`が実行される。同じクラスのメソッドなどはコードのようにクラス化してまとめておくと見やすくなる。また、`main()`が汚くならない。この状態だと VScode では各関数の近くにテスト実行ボタンが出てくる。それを押すことで一つずつでも実行できる。このファイルのテストを走らせるには

```
$ flutter test
```

またこのファイルを含むディレクトリ、あるいはこのファイル自体を指定してコマンドを実行しても問題ない。

```
$ flutter test test/unit_test
```

```
$ flutter test test/unit_test/controllers/counter_test.dart
```

を実行する。<br>
基本的には`test()`内の`expect()`で検証していく作業になる。`expect()`が失敗するとテストに失敗し、処理が中断されう。<br>

http.Client を mock 化するようなパッケージもあるのでそのようなデータが必要なときは[こちら](https://flutter.dev/docs/cookbook/testing/unit/mocking)

## Widget テスト

### 概要

Widget テストでは実際に Widget がどのような挙動を意識したテストができる。全ての Widget のテストコードを書く場合もあるがあまり現実的ではない(階層構造で重複も多い)。基本的には pages のファイルに対して widget_test/〇〇\_test.dart を生成しその中でテストする。

### 使用ツール

1. [flutter_test](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html)

### 手順

1. テストファイルの作成

2. テストの記述
   - `pumpWidget()` で Widget を描画
   - Finder で要素の取得
   - Matcher 要素に対して条件を指定する。
   - `expect()`で評価

### サンプルコード

#### _test/unit_test/controllers/counter_test.dart_

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:test_app/pages/home.dart';

void main() {
  group('HomePageTests', () {
    HomePageTests.testIncrement();
    HomePageTests.testDecrement();
  });
}

class HomePageTests {
  static void testIncrement() {
    testWidgets(
      'increment test.',
      (tester) async {
        await tester.pumpWidget(MaterialApp(home: HomePage()));
        expect(find.text('0'), findsOneWidget);
        expect(find.text('1'), findsNothing);

        await tester.tap(find.byIcon(Icons.add));
		// `tap`を反映した状態にしたいので再描画を試みている
        await tester.pump();

        expect(find.text('0'), findsNothing);
        expect(find.text('1'), findsOneWidget);
      },
    );
  }

  static void testDecrement() {
    testWidgets(
      'decrement test.',
      (tester) async {
        await tester.pumpWidget(MaterialApp(home: HomePage()));
        expect(find.text('0'), findsOneWidget);
        expect(find.text('1'), findsNothing);
        await tester.pump();
        expect(find.text('0'), findsOneWidget);
      },
    );
  }
}

```

ファイルの作り方などは単体テストの時と同じ。テスト本体の部分は`testWidgets()`の中に記述していく。テストしたい Widget を`pumpWidget()`でビルドするのがほとんどのケース。`pump()`は都度際描画するための関数。状態が変わった想定でもこの関数を呼ばないと反映されないので注意。`find`で要素を取得し、Matcher と呼ばれる findsOneWidget などの条件をその要素で検証して`expect()`で検証する。[取得の例](https://flutter.dev/docs/cookbook/testing/widget/finders)を公式がまとめてくれている。<br>
今回はタップだけの想定だったがドラッグや入力が Widget に対して想定されるときもある。公式にわかりやすい[例](https://flutter.dev/docs/cookbook/testing/widget/tap-drag)がある。

実行方法も単体テストと同じ。

## 結合テスト

### 概要

### 使用ツール

1. [flutter_test](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html)
2. [flutter_driver](https://api.flutter.dev/flutter/flutter_driver/flutter_driver-library.html)

### 手順

1.ディレクトリ、ファイルの作成

lib、test、とは別に test_driver というディレクトリを生成します。統合テストは、エミュレータ上のアプリとテストコードの 2 つが独立して相互に通信をしながらテストを行う。単体テストや Widget テストのように 1 つのプロセスの中で完結せず、2 つのファイルを準備して実行する必要がある。

```
test_driver
├── app.dart
└── app_test.dart

```

### サンプルコード

#### _test_driver/app.dart_

```dart
import 'package:flutter_driver/driver_extension.dart';
import 'package:test_app/main.dart' as app;

void main() {
	// 別プロセスのアプリケーションを起動できるようにするために拡張を有効にする
  enableFlutterDriverExtension();
  app.main();
}
```

このファイルでは実装したアプリを実行している。`flutter run`を実行した感じになる。

```dart
import 'package:flutter_driver/flutter_driver.dart';
import 'package:test/test.dart';

void main() {
  group('Counter App', () {
    final counterTextFinder = find.byValueKey('counter');
    final buttonFinder = find.byValueKey('increment');

    late FlutterDriver driver;

    setUpAll(() async {
		//初期設定
      driver = await FlutterDriver.connect();
    });

    tearDownAll(() async {
		//終わる時の処理
      await driver.close();
    });

    test('starts at 0', () async {
      expect(await driver.getText(counterTextFinder), '0');
    });

    test('increments the counter', () async {
      await driver.tap(buttonFinder);
      expect(await driver.getText(counterTextFinder), '1');
    });

    test('increments the counter during animation', () async {
      await driver.runUnsynchronized(() async {
        await driver.tap(buttonFinder);
        expect(await driver.getText(counterTextFinder), '1');
      });
    });
  });
}

```

テスト自体の書き方に大きな違いはない。今回はそのまま各バージョンも入れたかったのでシナリオをそのまま書いたが複数のシナリオがあるときはそれぞれクラス化する。<br>
実行するときは

```
$ flutter drive --target=test_driver/app.dart
```

スクロールが必要なときは公式の[こちら](https://flutter.dev/docs/cookbook/testing/integration/scrolling)を参照。
