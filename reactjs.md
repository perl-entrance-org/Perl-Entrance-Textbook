# はじめに

Webアプリケーションを開発するにあたり、エンドユーザが直感的な操作を行えるよう画面を遷移せずに書き換えたり、操作時間を短縮するためにAjaxで非同期通信を行ったりすることがあるかと思います。そういった機能はユーザ体験を大きく向上させることができ、ここ近年のWebアプリケーションではよりいっそう求められるようになってきました。

例えば、jQueryを使用して実装をしたとします。jQueryはDOM操作を簡単にしてくれるとても便利なライブラリです。最初のうちは問題なく使用できるでしょう。
がしかし、規模が拡大するにつれてコードは複雑さを増していきます。jQueryはアプリケーションのコードを整理するための枠組みは用意してくれません。データはDOMにあるため、テストも困難です。
そこで、規模の拡大とともに複雑化したコードを整理し、テストを行いやすくするため、JavaScriptフレームワークを導入する機会が増えてきました。

本資料では、数あるJavaScriptフレームワークの中でも最近注目されているReactとFluxを例に、JavaScriptフレームワークを用いたWebアプリケーション開発について学んでいきたいと思います。

# Reactとは

[React](https://reactjs.org/)とは、Facebook社とInstagram社が開発した **UIの構築に特化したJavaScriptライブラリ** です。FacebookやInstagramに加え、Yahoo!やAtlassianなど、[様々な企業](https://github.com/facebook/react/wiki/Sites-Using-React)で利用されはじめており、国内でも注目の高いライブラリです。

それでは、早速Reactの特徴を見ていきましょう。

## ビューに特化したライブラリ

「A JavaScript library for building user interfaces」とあるように、ReactはUIを構築するためのライブラリです。これは、MVCアーキテクチャでいうところのV（ビュー）の位置にあてはまります。よって、Reactはフレームワークではなく、既存のライブラリ・フレームワークと組み合わせて使用することができます。

また、React単体でも一定規模の動的なアプリケーションを構築することは可能です。

## 一方向のデータフロー

Reactでは、画面上のすべての要素を **コンポーネント** として定義します。
これは、画面上の要素を実際に線で囲ってみるとイメージしやすいと思います。

![コンポーネント](/reactjs/1.png)

上の図のように、画面上の各要素をコンポーネントとして定義します。
コンポーネントはHTMLと同様に階層化されて表現することが出来ます。

また、コンポーネントは基本的には状態を持たず（ **ステートレス** ）、レンダリングするために必要なデータはすべて上位のコンポーネントから渡されます。これにより、コンポーネント単体の再利用性を高め、管理しやすい状態を実現します。
また、コンポーネントのテストも行いやすくなります。

ただし、すべてのコンポーネントがステートレスになってしまうと静的なHTMLと変わりなくなってしまいます。そこで、Reactでは単純なコンポーネント群の上位にある重要なコンポーネントに状態を持たせます（ **ステートフル** ）。この状態をもとに、下位コンポーネント群にデータが渡されてレンダリングが行われます。ユーザから入力があった場合、上位コンポーネントに変更を伝えて状態を変更し、再レンダリングが行われます。

![コンポーネントツリーのレンダリング](/reactjs/2.png)

## Virtual DOM

コンポーネントの状態に変更があった際、下位コンポーネント群のレンダリングが行われます。これをそのまま画面に反映しようとすると、状態の変更がある度にDOMツリー全体を再構築する必要があるため、変更する必要がない箇所も都度書き換わってしまいます。
これは単純に処理コストも増えますし、ユーザー体験を大きく損なう可能性があります。

そこで、Reactでは **Virtual DOM** という仕組みでこの問題を解決しています。

すべてのコンポーネントがレンダリングされた際、その結果はVirtual DOMと呼ばれる仮想のDOMツリーに適用されます。仮想のDOMツリーはJavaScriptのメモリ上に存在し、画面とは切り離されて管理されています。そのため、仮想のDOMツリーへの適用は画面の表示に関する再計算等が発生せず、高速に動作します。
最終的に、Reactは仮想のDOMと実際のDOMを比較し、差分結果を自動的に実際のDOMへ適用します。

![Virtual DOMの仕組み](/reactjs/3.png)

この仕組みによって、細かなDOMの適用処理を考えなくて済むようになり、設計が単純化します。

# Reactを使ったTODOアプリの実装

さて、Reactの3つの大きな特徴を見てみました。
がしかし、読んだだけでは理解が進まないところがあるかもしれません。

そこで、ここからはシンプルなTODOアプリケーションを具体例に、Reactについてより深く学んでいきたいと思います。実装する内容は以下の通りです。

- すべてのTODOの表示
	- TODO名と作成日の一覧が表示される
- 新しいTODOの作成
	- フォームにTODO名を入力して作成できる
- 作成したTODOの削除
	- それぞれのTODOの削除ボタンから削除できる

また、完成したTODOアプリケーションのソースコードはGitHubにあります。

[https://github.com/perl-entrance-org/Perl-Entrance-Textbook/tree/examples/react-todo](/examples/react-todo)

## 準備

まずは、TODOアプリケーションを表示するためのHTMLドキュメントを作成しましょう。

```index.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>React TODO</title>
    <script src="https://unpkg.com/react@16/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@16/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone@7.0.0/babel.min.js"></script>
  </head>
  <body>
    <script type="text/babel">
      // ここにソースコードを書いていきます
    </script>
  </body>
</html>
```

ここでは、実装に必要なJavaScriptライブラリはすべてCDN経由で読み込むことにします。
また、実装は`<script>`タグの中に記述していきます。

## まずは表示してみる

では、Reactを使用して画面上に要素を表示してみましょう。

前述の通り、Reactでは、ページの構成要素をすべて **Reactコンポーネント** として定義します。
まずは、TODOアプリケーション全体を囲う要素となる「TodoAppコンポーネント」を定義・表示します。

```javascript:script
class TodoApp extends React.Component {
  render() {
    return (
      <div className="todoApp">
        Hello React! I am a TODO Application.
      </div>
    );
  }
}

ReactDOM.render(<TodoApp />, document.body);
```

上から順にコードを見ていきましょう。

まず、`React.Component`クラスを継承した`render`メソッドを持つクラスを作り、TodoAppコンポーネントを定義しています。renderメソッドは、コンポーネントが表示される際に呼ばれるメソッドです。必ず1つのコンポーネントを返す必要があります。
そして、定義したコンポーネントを、`ReactDOM.render`を用いて実際のDOM（今回は`<body>`タグ）に適用しています。

## JSXについて

ここで、コード上にあるXMLのような記述に目がいくかと思います。

ここに書かれている`<div>...</div>`や`<TodoApp />`は実際のHTML/XMLではありません。
Reactでは、 **JSX** というプリコンパイラが用意されており、XMLライクな簡易な記述でReactコンポーネントのインスタンスを作成することができます。このまま実行するとブラウザ側でエラーとなってしまうため、最終的にはXMLライクな記述を素のJavaScriptにプリコンパイルする必要があります。

もちろん、JSXを使用せず、最初から素のJavaScriptで記述することもできます。

```javascript:script
class TodoApp extends React.Component {
  render() {
    return (
      React.createElement('div', { className: 'todoApp' },
        'Hello, world! I am a TODO Application.'
      )
    );
  }
}

ReactDOM.render(React.createElement(TodoApp, null), document.body);
```

つまり、`<コンポーネント名 />`という記述は、`React.createElement('コンポーネント名', ...)`でインスタンスを作成するのと同じ結果になります。

JSXを使用するかどうかは開発者によって決めることができます。
この資料では、より手軽にReactコンポーネントを利用できるJSXの使用を前提に解説していきたいと思います。

## フォームとリストのスケルトン定義

続いて、先ほどのTodoAppコンポーネントと同様に、TODOを作成するための「TodoFormコンポーネント」とすべてのTODOを表示する「TodoListコンポーネント」を定義しましょう。

まだ中身は実装せず、仮の文字列が表示されるようにしておきます。

```javascript:script
class TodoForm extends React.Component {
  render() {
    return (
      <div className="todoForm">
        I am a TODO Form.
      </div>
    );
  }
}

class TodoList extends React.Component {
  render() {
    return (
      <div className="todoList">
        I am a TODO List.
      </div>
    );
  }
}
```

次に、作成した2つのコンポーネントを表示するため、TodoAppコンポーネントを修正します。

```javascript:script
class TodoApp extends React.Component {
  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm />
        <TodoList />
      </div>
    );
  }
}
```

TodoAppコンポーネントにTodoFormコンポーネントとTodoListコンポーネントが含まれ画面上に表示されました。

```コンポーネントの親子関係
TodoApp
  |
  |-- TodoForm
  `-- TodoList
```

このように、Reactではコンポーネントを組み立てて階層化し、画面の構成要素を表現していきます。

## TODOの表示

続いて、1つのTODOを表す「Todoコンポーネント」を定義しましょう。

親コンポーネントから渡されたTODO名と作成日を表示するようにしてみます。

```javascript:script
class Todo extends React.Component {
  render() {
    return (
      <div className="todo">
        <span className="name">{this.props.children}</span>
        <span className="date">{this.props.created_at}</span>
      </div>
    );
  }
}
```

親から子へのデータの受け渡しは、コンポーネントの **props** （プロパティ）経由で行います。
propsは`this.props`でアクセスでき、`this.props.children`がTODO名、`this.props.created_at`が作成日として、表示されています。
（JSXの中では、波括弧 `{}` を用いてJavaScriptのコードを記述することができます。）

では、TodoListコンポーネントにいくつかのTodoコンポーネントを追加してみましょう。

```javascript:script
class TodoList extends React.Component {
  render() {
    return (
      <div className="todoList">
        <Todo created_at="2015/05/01 9:00:00">牛乳を買う</Todo>
        <Todo created_at="2015/05/01 9:00:00">パンを買う</Todo>
      </div>
    );
  }
}
```

親のTodoListコンポーネントから子のTodoコンポーネントにいくつかのデータを渡しています。
例えば、「2015/05/01 9:00:00」をcreated_at属性に、「牛乳を買う」を子ノードに指定して、1つめのTodoコンポーネントに渡しています。

このように、Reactでは親コンポーネントから渡されたデータを元にレンダリングを行い、コンポーネントをステートレスに保ちます。

## TODOの動的表示

先ほどのコードではTodoListコンポーネントの中にTodoコンポーネントをハードコードしていました。これでは静的なHTMLと全く変わりがありません。
そこで、最上位のコンポーネントであるTodoAppコンポーネントの **state** （状態）としてデータを持たせ、それを使ってTODOを動的に表示するようにしましょう。

まずは、TodoAppコンポーネントのstateの初期値を定義しましょう。

```javascript:script
class TodoApp extends React.Component {
  constructor(...args) {
    super(...args);
    
    this.state = {
      todos: []
    };
  }
  
  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm />
        <TodoList />
      </div>
    );
  }
}
```

`constructor`メソッドで`this.state`の初期化を行い、コンポーネントのstateの初期値を決めています。

次に、先ほどハードコードしていたTODOの内容をstateに設定します。

```javascript:script
class TodoApp extends React.Component {
  constructor(...args) {
    super(...args);

    this.state = {
      todos: []
    };
  }

  componentDidMount() {
    // NOTE: ここでfetch APIを用いてサーバサイドから取得してもよい
    const todos = [
      { id: 'i9tajxy9', name: '牛乳を買う', created_at: '2015/05/01 9:00:00' },
      { id: 'i9ta58tx', name: 'パンを買う', created_at: '2015/05/01 9:00:00' }
    ];

    this.setState({ todos });
  }

  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm />
        <TodoList todos={this.state.todos} />
      </div>
    );
  }
}
```

ここで、`componentDidMount`メソッドを新しく定義しています。
componentDidMountメソッドは、コンポーネントがレンダリングされたときに実行されます。
コンポーネントには **ライフサイクルが** あり、他にもいくつかのメソッドを定義することで、状態の変化に応じて実行されます。

次に、TODOの内容を一度`todos`変数に格納しています。
今回はサーバサイドとの通信は行いませんが、一般的なアプリケーションの様にfetch APIを用いてサーバサイドからデータを取得してもよいでしょう。
また、それぞれのTODOを特定できるよう、ランダムな`id`を新たに割り振っています。

componentDidMountメソッドの最後で、todosを`setState`でstateに格納しています。
Reactでは、setStateが呼ばれることで再レンダリングが行われるため、データの更新は必ずsetStateで行いましょう。

そして、格納された`this.state.todos`をTodoListコンポーネントにprops経由で渡しています。

では、TODOを動的に表示してみましょう。

```javascript:script
class TodoList extends React.Component {
  render() {
    var todos = this.props.todos.map(todo => (
      <Todo key={todo.id} created_at={todo.created_at}>{todo.name}</Todo>
    ));
    return (
      <div className="todoList">
        {todos}
      </div>
    );
  }
});
```

親コンポーネントから渡されたtodosを繰り返し処理し、Todoコンポーネントを組み立てています。

ここで、新たにTodoコンポーネントに`key`を指定しています。
TodoListコンポーネントのような同じコンポーネントを複数並べる構成の場合、`key`にそれぞれユニークな値を与える必要があります。keyを指定することによって、ReactはVirtual DOMと実際のDOMを比較しやすくなり、最小限の変更だけ実際のDOMに適用することができるからです。

このようにして、Reactでは最上位のコンポーネントにstateを持たせ、それに基づいて下位コンポーネント群の動的な組み立てを行います。

## TODOの作成

ここまででTODOの表示を行うことができました。
今度は、フォーム上から新しいTODOを作成できるようにしてみましょう。

TodoFormコンポーネントにTODO名の入力欄と送信ボタンを定義していきます。

```javascript:script
class TodoForm extends React.Component {
  render() {
    return (
      <form className="todoForm">
        <input type="text" placeholder="TODOを入力..." />
        <button type="submit">作成</button>
      </form>
    );
  }
}
```

シンプルなフォームが表示されました。

次に、フォームのsubmitイベントをハンドリングできるようにします。

```javascript:script
class TodoForm extends React.Component {
  constructor(...args) {
    super(...args);
    
    this.handleSubmit = this.handleSubmit.bind(this);
  }

  handleSubmit(e) {
    e.preventDefault();
  }

  render() {
    return (
      <form className="todoForm" onSubmit={this.handleSubmit}>
        <input type="text" placeholder="TODOを入力..." />
        <button type="submit">作成</button>
      </form>
    );
  }
}
```

Reactでは、`on + イベント名`属性でイベントハンドラをコンポーネントに結びつけることができます。ここでは、フォームのsubmitイベントが発生すると`handleSubmit`メソッドが実行されます。
また、ブラウザのデフォルトアクションを抑止するため、イベントオブジェクトの`preventDefault()`を呼び出しています。（このあたりは、jQueryをはじめ、他ライブラリ・フレームワークと同様に考えることができます。）

このままだと、作成ボタンを押しても入力値が入力エリアに残ってしまいます。値をクリアしてあげましょう。

```javascript:script
class TodoForm extends React.Component {
  constructor(...args) {
    super(...args);

    this.nameRef = React.createRef();
    this.handleSubmit = this.handleSubmit.bind(this);
  }

  handleSubmit(e) {
    e.preventDefault();

    const name = this.nameRef.current;
    name.value = '';
  }

  render() {
    return (
      <form className="todoForm" onSubmit={this.handleSubmit}>
        <input type="text" placeholder="TODOを入力..." ref={this.nameRef} />
        <button type="submit">作成</button>
      </form>
    );
  }
}
```

`React.createRef`を使い、`this.nameRef`にコンポーネントへの参照を割り当てます。`React.createRef`で作られたコンポーネントの参照は`ref`属性を子コンポーネントに追加することでコンポーネントと紐付けられます。
さらに、そのコンポーネントに紐付く実際のDOMを`this.nameRef.current`で取得して`name`変数に格納しています。
最後に、`name.value`を空にすることでフォームに入力した値がクリアされます。

では、入力された内容をもとに、TODOを作成していきましょう。

```javascript:script
class TodoApp extends React.Component {
  constructor(...args) {
    super(...args);

    this.state = {
      todos: []
    };
    
    this.todoCreate = this.todoCreate.bind(this);
  }

  componentDidMount() {
    // NOTE: ここでfetch APIを用いてサーバサイドから取得してもよい
    const todos = [
      { id: 'i9tajxy9', name: '牛乳を買う', created_at: '2015/05/01 9:00:00' },
      { id: 'i9ta58tx', name: 'パンを買う', created_at: '2015/05/01 9:00:00' }
    ];

    this.setState({ todos });
  }

  todoCreate(name) {
    // TODO: ここでTODOの作成処理を行う
  }

  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm onTodoCreate={this.todoCreate} />
        <TodoList todos={this.state.todos} />
      </div>
    );
  }
}
```

TODOのデータはTodoAppコンポーネントのstateとして持っているため、TodoAppコンポーネント自身がTODOを作成するのがよいでしょう。そこで、TODOを作成するための`todoCreate`メソッドを定義し、子コンポーネントから実行してもらうことにします。
TODOの表示と同様に、イベントハンドラについてもprops経由で親から子コンポーネントに渡します。

渡されたTodoForm側を見てみましょう。

```javascript:script
class TodoForm extends React.Component {
  constructor(...args) {
    super(...args);

    this.nameRef = React.createRef();
    this.handleSubmit = this.handleSubmit.bind(this);
  }

  handleSubmit(e) {
    e.preventDefault();

    var name = this.nameRef.current;
    if (name.value !== '') {
      this.props.onTodoCreate(name.value.trim());
    }
    name.value = '';
  }

  render() {
    return (
      <form className="todoForm" onSubmit={this.handleSubmit}>
        <input type="text" placeholder="TODOを入力..." ref={this.nameRef} />
        <button type="submit">作成</button>
      </form>
    );
  }
}
```

TodoFormコンポーネントでは、入力された内容が空ではない場合に、propsの`onTodoSubmit`を呼び出しています。すなわち、TodoAppコンポーネントの`todoCreate`メソッドです。

最後に、TODOの作成処理の実装です。

```javascript:script
class TodoApp extends React.Component {
  constructor(...args) {
    super(...args);

    this.state = {
      todos: []
    };
    
    this.todoCreate = this.todoCreate.bind(this);
  }

  componentDidMount() {
    // NOTE: ここでfetch APIを用いてサーバサイドから取得してもよい
    const todos = [
      { id: 'i9tajxy9', name: '牛乳を買う', created_at: '2015/05/01 9:00:00' },
      { id: 'i9ta58tx', name: 'パンを買う', created_at: '2015/05/01 9:00:00' }
    ];

    this.setState({ todos });
  }

  todoCreate(name) {
    // NOTE: ここでfetch APIを用いてサーバサイドに送信・作成してもよい
    const todo = {
      id: (Date.now() + Math.floor(Math.random() * 999999)).toString(36),
      name,
      created_at: (new Date()).toLocaleString()
    };

    this.setState(state => ({
      todos: state.todos.concat([todo])
    }));
  }

  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm onTodoCreate={this.todoCreate} />
        <TodoList todos={this.state.todos} />
      </div>
    );
  }
}
```

`todoCreate`メソッドでは、渡された名前を元にTODOのデータを作成し、`todo`変数に格納しています。（`id`は、ランダムな文字列を取得して指定しています。`created_at`は現在の日付から人間が読みやすい文字列に変換しています。）
そして、`this.state`を破壊しないよう、`concat`で新しい配列を作成して`setState`メソッドでstateを更新しています。

一見すると、`todos`が新しい配列になり、画面に表示されているTODO群がすべて再レンダリングされるように感じるかもしれません。Reactでは、Virtual DOMによる差分適用が自動的にされるため、todos全体を更新しても新しく作成したTODOのみ実際のDOMツリーに追加されます。

このように、Reactでは実際のDOMを意識することなくデータを富豪的に更新することができるのです。

## TODOの削除

いよいよ最後の実装になります。作成されたTODOを削除できるボタンを追加しましょう。

まずはTodoコンポーネントに削除ボタンを定義していきます。
先ほどのフォームと同様に、イベントハンドラも定義します。

```javascript:script
class Todo extends React.Component {
  constructor(...args) {
    super(...args);

    this.handleDestroy = this.handleDestroy.bind(this);
  }

  handleDestroy(e) {
    e.preventDefault();
  }

  render() {
    return (
      <div className="todo">
        <span className="name">{this.props.children}</span>
        <span className="date">{this.props.created_at}</span>
        <button onClick={this.handleDestroy}>削除</button>
      </div>
    );
  }
}
```

まだなにも動作しない削除ボタンが定義できました。

続いて削除処理の実装です。

```javascript:script
class TodoApp extends React.Component {
  constructor(...args) {
    super(...args);

    this.state = {
      todos: []
    };

    this.todoCreate = this.todoCreate.bind(this);
    this.todoDestroy = this.todoDestroy.bind(this);
  }

  componentDidMount() {
    // NOTE: ここでfetch APIを用いてサーバサイドから取得してもよい
    const todos = [
      { id: 'i9tajxy9', name: '牛乳を買う', created_at: '2015/05/01 9:00:00' },
      { id: 'i9ta58tx', name: 'パンを買う', created_at: '2015/05/01 9:00:00' }
    ];

    this.setState({ todos });
  }

  todoCreate(name) {
    // NOTE: ここでfetch APIを用いてサーバサイドに送信・作成してもよい
    const todo = {
      id: (Date.now() + Math.floor(Math.random() * 999999)).toString(36),
      name,
      created_at: (new Date()).toLocaleString()
    };

    this.setState(state => ({
      todos: state.todos.concat([todo])
    }));
  }

  todoDestroy(id) {
    // TODO: ここでTODOの削除処理を行う
  }

  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm onTodoCreate={this.todoCreate} />
        <TodoList todos={this.state.todos} onTodoDestroy={this.todoDestroy} />
      </div>
    );
  }
}
```

作成時と同様、TodoAppコンポーネント自身がTODOを削除するのがよいでしょう。`todoDestroy`メソッドを定義し、TodoListコンポーネントを経由してTodoコンポーネントに渡します。

TodoListコンポーネントを見てみましょう。

```javascript:script
class TodoList extends React.Component {
  render() {
    const todos = this.props.todos.map(todo => (
      <Todo key={todo.id} id={todo.id} created_at={todo.created_at} onTodoDestroy={this.props.onTodoDestroy}>{todo.name}</Todo>
    ));
    return (
      <div className="todoList">
        {todos}
      </div>
    );
  }
}
```

ここでは受け取ったイベントハンドラをそのままTodoコンポーネントに渡しています。
また、TODOの削除は`id`で判別するため、新たに`id`を渡しています。

Todoコンポーネントを見てみましょう。

```javascript:script
class Todo extends React.Component {
  constructor(...args) {
    super(...args);

    this.handleDestroy = this.handleDestroy.bind(this);
  }

  handleDestroy(e) {
    e.preventDefault();

    this.props.onTodoDestroy(this.props.id);
  }

  render() {
    return (
      <div className="todo">
        <span className="name">{this.props.children}</span>
        <span className="date">{this.props.created_at}</span>
        <button onClick={this.handleDestroy}>削除</button>
      </div>
    );
  }
}
```

TodoListコンポーネントから渡された`onTodoDestroy`イベントハンドラを実行しています。
これで、TODOの作成と同様に、子コンポーネントから親のstateを更新しています。

最後に、TodoAppで削除しましょう。

```javascript:script
class TodoApp extends React.Component {
  constructor(...args) {
    super(...args);

    this.state = {
      todos: []
    };

    this.todoCreate = this.todoCreate.bind(this);
    this.todoDestroy = this.todoDestroy.bind(this);
  }

  componentDidMount() {
    // NOTE: ここでfetch APIを用いてサーバサイドから取得してもよい
    const todos = [
      { id: 'i9tajxy9', name: '牛乳を買う', created_at: '2015/05/01 9:00:00' },
      { id: 'i9ta58tx', name: 'パンを買う', created_at: '2015/05/01 9:00:00' }
    ];

    this.setState({ todos });
  }

  todoCreate(name) {
    // NOTE: ここでfetch APIを用いてサーバサイドに送信・作成してもよい
    const todo = {
      id: (Date.now() + Math.floor(Math.random() * 999999)).toString(36),
      name,
      created_at: (new Date()).toLocaleString()
    };

    this.setState(state => ({
      todos: state.todos.concat([todo])
    }));
  }

  todoDestroy(id) {
    // NOTE: ここでfetch APIを用いてサーバサイドに送信・削除してもよい
    const newTodos = this.state.todos.filter(todo => todo.id !== id);

    this.setState({ todos: newTodos });
  }

  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm onTodoCreate={this.todoCreate} />
        <TodoList todos={this.state.todos} onTodoDestroy={this.todoDestroy} />
      </div>
    );
  }
}
```

stateの`todos`を`filter`メソッドで削除対象以外のものに絞り込みをして新しい配列を`newTodos`に格納しています。あとは、TODOの作成と同様に、`setState`メソッドでstateを更新するだけです。

お疲れ様でした。以上でTODOアプリケーションの実装が完了しました！

## 実装してみて

Reactを使うと、DOMを意識することなく画面の各要素を組み立てて表示できることがわかりました。表示処理は明示的に行う必要はなく、すべてReact任せです。また、各コンポーネントは渡ってきたデータをもとに表示するだけなので、とてもシンプルでテストしやすい状態になります。

一方、以下の点が気になると思います。

- TODOの削除を行うためのイベントハンドラを、TodoApp⇒TodoList⇒Todo、のように順に引き継いでいかなければならない
- TODOの追加やTODOの削除などのビジネスロジックがTodoAppコンポーネントに集中してしまう

これでは、アプリケーションの要件が増えるにつれてメンテナンス性が悪くなってしまいますね。
ReactはあくまでUIに特化したライブラリのため、これらの問題に対する解決策は提示していません。

ではどうしたらよいでしょうか。
そこで、 **Flux** の登場です。

# Fluxとは

[Flux](https://facebook.github.io/flux/)とは、Facebook社が提唱しているアプリケーションのアーキテクチャです。これはライブラリ・フレームワークではなく、MVC等と同じく、アプリケーションの設計手法の一つとなります。

では、Fluxでのアプリケーションの設計思想は一体どのようなものになるのでしょうか。
早速、Fluxの特徴を見ていきましょう。

## 一方向のデータフロー

Fluxでは、Reactと同様にアプリケーションの複雑さをなくすためデータの流れを一方向に定めます。

```
                                                       +------------+                  
                                      +-------+        |            |     +---------+  
                                      |Actions+--------> Dispatcher +----->Callbacks|  
                                      +---^---+        |            |     +----+----+  
                                          |            +------------+          |       
                                          |                                    |       
+---------+   +---------------+   +-------+--------+                       +---v---+   
|         +--->               +--->                |                       |       |   
| Web API |   | Web API Utils |   | Action Creator |                       | Store |   
|         <---+               <---+                |                       |       |   
+---------+   +---------------+   +-------^--------+                       +---+---+   
                                          |                                    |       
                                          |            +------------+          |       
                                 +--------+--------+   |            |   +------v-------
                                 |User Interactions<---+ React View <---+Change Events|
                                 +-----------------+   |            |   |Store Queries|
                                                       +------------+   +-------------+
```

上の図を見ると、データが常に一方向に流れていることがわかります。
そのため、全体の処理の流れが把握しやすく、ある程度の規模になってもコードが複雑化しにくい設計になっています。

## Fluxの本質的な4つの要素

## Action Creators

Action Creatorsは、その名前のとおりActionを作成します。
Actionは、そのActionを特定するための識別子と実行する際に必要なデータがひとまとめになったものになります。Actionは、後述するDispatcherによって、Dispatcherに登録されているStoreに渡され実行されます。

また、必要であれば外部APIに対してのリクエストもここで行い、その結果をActionとして作成します。

## Dispatcher

Dispatcherは、Action Creatorsから渡されたActionを、登録されているStoreのコールバック関数に渡します。これはpub/subに似ていますが、イベントの種別を管理する必要がないため、とても単純です。
また、DispatcherはStoreのコールバック関数にActionを渡す際、Store間の依存関係を解決する役目を持ちます。

## Store

Storeは、Dispatcherから渡されたActionを元にビジネスロジックを実行し、作成・更新されたデータを管理します。データの更新は外部から行うことはできません。必ずDispatcher経由で渡されたActionを元に更新処理を行う必要があります。
データの参照については、外部から行うことは可能です。また、データに変更があった際にchangeイベントを発行します。Viewは、そのイベントをトリガーにデータの参照を行います。

## View

Viewは、ユーザから何らかの入力があった際、Action Creator経由でアクションを発行します。また、Storeからデータを取得し表示します。このViewのことを、特別に **Controller-View** と呼びます。
先ほどのTODOアプリケーションの場合、Controller-ViewはTodoAppコンポーネントにあたります。単純なコンポーネント群の上位にある重要なViewをController-Viewとして定義します。

ViewはReact以外でも問題はありませんが、FacebookはReactの利用を推奨しています。

## Fluxの実装

Fluxはアーキテクチャであり、アプリケーション設計の考え方を示します。
この考え方に基づき、様々な実装が存在しています。

- https://github.com/facebook/flux
- https://github.com/acdlite/flummox
- https://github.com/mizchi/arda
- etc...

Facebookが提供しているFluxの実装は、Dispatcherのみのミニマムな実装です。
そもそも、Fluxの考え方に基づいて実装すること自体はそこまで難しくはありません。必要に応じて自前で用意することもできるでしょう。

# Fluxを使ったTODOアプリの再実装

さて、Fluxの考え方については見えてきたものの、実際にどんな実装を行っていくのでしょうか。
ここからは、先ほどReactで実装したTODOアプリケーションをFacebookのFlux実装で再実装し、Fluxについてより深く学んでいきたいと思います。

また、完成したTODOアプリケーションのソースコードはGitHubにあります。

https://github.com/miniturbo/flux-todo

## 準備

先ほどまではCDN経由で実装に必要なJavaScriptライブラリを使用していましたが、再実装を行うにあたって必要なライブラリ（モジュール）が増えるため、Node.jsを導入してモジュールを管理・使用したいと思います。
また、Reactコンポーネントについても、Node.jsの機構に従っていくつかのモジュールに分割して管理しやすいようにしたいと思います。

実際には、Node.jsではなくブラウザで動作するため、ブラウザでもモジュールを扱えるようにする必要があります。本資料では詳しくは触れませんが、[Browserify](http://browserify.org/)を用いてNode.jsのようにモジュールとして扱えるようにします。

## Node.jsのインストール

まずは、[Node.js](https://nodejs.org/)のインストールを行いましょう。

Homebrewを利用している方は、`brew`コマンドでNode.jsをインストールしましょう。

```console:Homebrewの場合
% brew install node
```

ndenvを利用している方は、`ndenv`コマンドでNode.jsをインストールしましょう。

```console:ndenvの場合
% ndenv install v8.11.4
% ndenv global v8.11.4
```

これでNode.jsの導入ができました。

## 必要なモジュールのインストール

次に、必要なモジュールのインストールを行いましょう。
モジュール、Node.jsのパッケージマネージャの **npm** で、インストール・管理を行います。

ここでは、下記の`package.json`を新たに作成し、`npm install`でモジュールをインストールします。

```package.json
{
  "name": "flux-todo",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "build": "browserify -t reactify ./js/app.js > ./js/bundle.js",
    "watch": "watchify -v -t reactify ./js/app.js -o ./js/bundle.js"
  },
  "author": "YOUR NAME <YOUR EMAIL ADDRESS>",
  "license": "MIT",
  "dependencies": {
    "flux": "^2.0.3",
    "keymirror": "^0.1.1",
    "object-assign": "^2.0.0",
    "react": "^16.5.0",
    "react-dom": "^16.5.0"
  },
  "devDependencies": {
    "browserify": "^10.2.0",
    "reactify": "^1.1.1",
    "watchify": "^3.2.1"
  }
}
```

```console:モジュールのインストール
% npm install
```

## ディレクトリの作成

最後に、必要なディレクトリを作成しましょう。

```console:ディレクトリの作成
% mkdir -p js/{actions,components,constants,dispatcher,stores}
```

以上で準備は整いました。
ここまでで、下記のようなファイル・ディレクトリ構成になっていると思います。

```shell-session:ファイル・ディレクトリ構成
% tree -A -L 2
.
+-- README.md
+-- index.html
+-- js
|   +-- actions
|   +-- components
|   +-- constants
|   +-- dispatcher
|   +-- stores
+-- node_modules
|   +-- browserify
|   +-- flux
|   +-- keymirror
|   +-- object-assign
|   +-- react
|   +-- reactify
|   +-- watchify
+-- package.json
```

## Dispatcherの実装

それでは、まずはじめにDispatcherを実装しておきましょう。
Dispatcherは、Fluxの4つの要素のうちAction CreatorsとStoreを仲介する中心的な存在です。

といっても、Dispatcherの実装は驚くほどシンプルです。Facebookが提供するDispatcherをそのまま利用します。

```js/dispatcher/AppDispatcher.js
const Dispatcher = require('flux').Dispatcher;

module.exports = new Dispatcher();
```

`require`で`flux`モジュールを読み込んでいます。
作成したDispatcherインスタンスは、外部で使用できるよう`module.exports`に代入しています。

これでDispatcherの実装は完了です。次に進みましょう。

## Actionの実装

続いて、TODOアプリケーションのアクションをActionとして定義しましょう。
「TODOの追加」「TODOの削除」に加え、`componentDidMount`で行っていた`todos`の初期化を「TODOのセットアップ」としてそれぞれ定義します。

まずはActionの識別子から定義しましょう。

```js/constants/TodoConstants.js
const keyMirror = require('keymirror');

module.exports = keyMirror({
  TODO_SETUP:   null,
  TODO_CREATE:  null,
  TODO_DESTROY: null
});
```

keymirrorは、渡されたオブジェクトのkeyを対応するvalueにコピーするだけのモジュールです。
上記コードは以下のようなオブジェクトになります。

```keyMirrorの結果
{
  TODO_SETUP:   'TODO_SETUP',
  TODO_CREATE:  'TODO_CREATE',
  TODO_DESTROY: 'TODO_DESTROY'
}
```

この識別子のkeyとvalueを使ってActionとStoreの紐付けが行われます。

続いてAction Creatorsを実装しましょう。

```js/actions/TodoActions.js
const AppDispatcher = require('../dispatcher/AppDispatcher');
const TodoConstants = require('../constants/TodoConstants');

module.exports = {
  setup: function() {
    // NOTE: ここでAjaxを用いてサーバサイドから取得してもよい
    var todos = [
      { id: 'i9tajxy9', name: '牛乳を買う', created_at: '2015/05/01 9:00:00' },
      { id: 'i9ta58tx', name: 'パンを買う', created_at: '2015/05/01 9:00:00' }
    ];
    AppDispatcher.dispatch({
      actionType: TodoConstants.TODO_SETUP,
      todos: todos
    });
  },

  create: function(name) {
    // NOTE: ここでAjaxを用いてサーバサイドから取得・作成してもよい
    AppDispatcher.dispatch({
      actionType: TodoConstants.TODO_CREATE,
      name: name
    });
  },

  destroy: function(id) {
    // NOTE: ここでAjaxを用いてサーバサイドから取得・削除してもよい
    AppDispatcher.dispatch({
      actionType: TodoConstants.TODO_DESTROY,
      id: id
    });
  }
};
```

ここでは、`setup`、`create`、`destroy`メソッドを定義し、外部から実行できるように`module.exports`で提供しています。それぞれの中では、アクションを実行するための値を用意し、最初に実装した`AppDispatcher`の`dispatch`メソッドを実行しています。
dispatchメソッドに渡した`actionType`に応じて、Store側でビジネスロジックの実行を判断することになります。

## Storeの実装

続いて、Storeの実装に移ります。
先ほどのAction Creatorsから渡されたActionをもとに、ビジネスロジックの実行とデータの更新を行います。

```js/stores/TodoStore.js
const EventEmitter  = require('events').EventEmitter;
const assign        = require('object-assign');
const AppDispatcher = require('../dispatcher/AppDispatcher');
const TodoConstants = require('../constants/TodoConstants');

var _todos = [];

function setup(todos) {
  _todos = todos;
}

function create(name) {
  var todo = {
    id: (Date.now() + Math.floor(Math.random() * 999999)).toString(36),
    name: name,
    created_at: (new Date()).toLocaleString()
  };
  _todos = _todos.concat([todo]);
}

function destroy(id) {
    var newTodos = _todos.filter(function(todo) { return todo.id == id ? false : true });
    _todos = newTodos;
}

var TodoStore = assign({}, EventEmitter.prototype, {
  getAll: function() {
    return _todos;
  },

  emitChange: function() {
    this.emit('change');
  },

  addChangeListener: function(callback) {
    this.on('change', callback);
  },

  removeChangeListener: function(callback) {
    this.removeListener('change', callback);
  }
});

AppDispatcher.register(function(action) {
  switch(action.actionType) {
    case TodoConstants.TODO_SETUP:
      setup(action.todos);
      TodoStore.emitChange();
      break;

    case TodoConstants.TODO_CREATE:
      var name = action.name.trim();
      if (name !== '') {
        create(name);
        TodoStore.emitChange();
      }
      break;

    case TodoConstants.TODO_DESTROY:
      destroy(action.id);
      TodoStore.emitChange();
      break;

    default:
      // noop
  }
});

module.exports = TodoStore;
```

少しばかり長いですが、やっていることは単純です。

`setup`、`create`、`destroy`関数は、`TodoStore`のビジネスロジックです。処理内容は、TodoAppコンポーネントに定義していたものと同じです。このようにビジネスロジックをStoreに分離することができます。このビジネスロジック群は、下の方にある`AppDispatcher.register`のコールバック関数で実行されます。

続いて、TodoStore自体の定義です。TodoStoreは`EventEmitter`の機能を持っています。TodoStoreは外部に公開され、主にController-Viewが使用することになります。
詳しくはController-View側で見ていくことにしましょう。

最後に、`AppDispatcher.register`にコールバック関数を登録しています。
前述のAction CreatorsがAppDispatcherの`dispatch`を読んだ際、Dispatcherはこのコールバック関数を実行することになります。Actionが渡されるので、`TodoConstants`でActionの識別子を確認し、それぞれのビジネスロジックを実行します。データが更新されたあとに、`TodoStore.emitChange`メソッドでchangeイベントを発行します。

## Viewの分割

続いて、先ほどまで実装していたViewの各コンポーネントをそれぞれのモジュールに分割し、再実装しましょう。

まずはTodoコンポーネントです。

```js/components/Todo.js
const React       = require('react');
const TodoActions = require('../actions/TodoActions');

class Todo extends React.Component {
  constructor(...args) {
    super(...args);
    
    this.handleDestroy = this.handleDestroy.bind(this);
  }

  handleDestroy(e) {
    e.preventDefault();

    TodoActions.destroy(this.props.id);
  },

  render() {
    return (
      <div className="todo">
        <span className="name">{this.props.children}</span>
        <span className="date">{this.props.created_at}</span>
        <button onClick={this.handleDestroy}>削除</button>
      </div>
    );
  }
}

module.exports = Todo;
```

先ほどまでは親から渡された`onTodoDestroy`イベントハンドラでTODOの削除を行っていましたが、Action Creatorsである`TodoActions`の`destroy`メソッドで削除を行えるようになります。

続いて、TodoListコンポーネントです。

```js/components/TodoList.js
const React = require('react');
const Todo  = require('./Todo');

class TodoList extends React.Component {
  render() {
    const todos = this.props.todos.map(todo => (
      <Todo key={todo.id} id={todo.id} created_at={todo.created_at}>{todo.name}</Todo>
    ));
    return (
      <div className="todoList">
        {todos}
      </div>
    );
  }
}

module.exports = TodoList;
```

`Todo`コンポーネントで直接アクションできるようになったため、`onTodoDestroy`イベントハンドラを引き回す必要がなくなりました。

続いて、TodoFormコンポーネントです。

```js/components/TodoForm.js
const React       = require('react');
const TodoActions = require('../actions/TodoActions');

class TodoForm extends React.Component {
  constructor(...args) {
    super(...args);

    this.nameRef = React.createRef();
    this.handleSubmit = this.handleSubmit.bind(this);
  }

  handleSubmit(e) {
    e.preventDefault();

    var name = this.nameRef.current;
    TodoActions.create(name.value.trim());
    name.value = '';
  },

  render() {
    return (
      <form className="todoForm" onSubmit={this.handleSubmit}>
        <input type="text" placeholder="TODOを入力..." ref={this.nameRef} />
        <button type="submit">作成</button>
      </form>
    );
  }
}

module.exports = TodoForm;
```

こちらも、Todoコンポーネントと同様に直接`TodoActions`経由でTODOの作成を行えるようになりました。

最後に、TodoAppコンポーネントです。

```js/components/TodoApp.js
const React       = require('react');
const TodoActions = require('../actions/TodoActions');
const TodoStore   = require('../stores/TodoStore');
const TodoForm    = require('./TodoForm');
const TodoList    = require('./TodoList');

class TodoApp extends React.Component {
  constructor(...args) {
    super(...args);

    this.state = {
      todos: TodoStore.getAll()
    };
  }

  componentDidMount() {
    TodoStore.addChangeListener(this._onChange);
    TodoActions.setup();
  }

  componentWillUnmount: function() {
    TodoStore.removeChangeListener(this._onChange);
  }

  _onChange() {
    this.setState({ todos: TodoStore.getAll() });
  }

  render() {
    return (
      <div className="todoApp">
        <h1>TODO Application</h1>
        <TodoForm />
        <TodoList todos={this.state.todos} />
      </div>
    );
  }
}

module.exports = TodoApp;
```

ずいぶんとすっきりしましたね。
TODOのデータ更新のロジックをすべて分離することができ、イベントハンドラを末端のコンポーネントまで伝える必要がなくなりました。

コンポーネントのライフサイクルに合わせていくつか注目して見てみましょう。

TodoAppコンポーネントが作成された際、`getInitialState`メソッドが実行されます。ここでは、`TodoStore.getAll`メソッドで、Storeにある初期値を取得しstateに指定しています。

TodoAppコンポーネントが実際のDOMツリーに追加された際、`componentDidMount`メソッドが実行されます。ここでは、`TodoStore.addChangeListener`メソッドで、Storeのデータが更新された際に実行してほしいコールバック関数を登録しています。つまり、Storeのデータが更新される度、`_onChange`メソッドが実行され、`setState`でstateが更新されます。

TodoAppコンポーネントが実際のDOMツリーから削除された際、`componentWillUnmount`メソッドが実行されます。メモリリークを割けるため、` TodoStore.removeChangeListener`メソッドで登録していたコールバック関数を解除しています。

## アプリの表示

最後に、TodoAppコンポーネントを実際のDOMツリーにレンダリングしましょう。

```js/app.js
const React    = require('react');
const ReactDOM = require('react-dom');
const TodoApp  = require('./components/TodoApp');

ReactDOM.render(<TodoApp />, document.body);
```

このファイルをBrowserifyでブラウザが解釈できるJavaScriptに変換します。

```console:JavaScriptのビルド
% npm run-script build
```

このコマンドを実行することで、`require`したすべてのモジュールを1枚のファイルに連結し`bundle.js`として保存します。同時に、ReactのJSXの文法も素のJavaScriptに変換されています。

あとはindex.htmlを修正すれば完了です。

```index.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>React TODO</title>
  </head>
  <body>
    <script src="./js/bundle.js"></script>
  </body>
</html>
```

お疲れ様でした。以上でTODOアプリケーションの再実装が完了しました！

## 実装してみて

Reactで実装を行った際に抱えていた悩みはいずれも解決することができました。

- TODOの削除を行うためのイベントハンドラは引き継ぐ必要がなくなり、直接Action Creators経由でアクションを実行することができた
- TODOの追加やTODOの削除などのビジネスロジックをStoreに適切に切り出すことができた

このように、Fluxを使うと、それぞれのコードを適切に分割し、データフローを一方向に定められることがわかりました。これによってアプリケーションの規模が拡大しても、スケール性を保つことができます。

# まとめ

いかがでしたでしょうか。
ReactとFluxを例に取りながら、JavaScriptフレームワークを用いたWebアプリケーション開発について学んできました。

それなりに規模のある動的なWebアプリケーションを開発する場合、JavaScriptフレームワークは必要不可欠な存在です。今回のReactとFluxを使った例のように、スケール性を保ちつつもシンプルに実装していけることが望まれます。

また、今回はReactとFluxを選定しましたが、別のJavaScriptフレームワークに目を向けてみるのもいいかもしれません。小規模向けのVue.js/Knockout.jsやフルスタックフレームワークのAngularJS、ドキュメントが豊富に揃っているBackbone.jsなど、いくつものJavaScriptフレームワークが存在します。
選定には、開発するWebアプリケーションの特性やチームメンバーのスキルなど、様々な観点が必要となってくるでしょう。

本資料をきっかけに、周りの開発者やチームメンバーと意見を交わし、よりJavaScriptフレームワークに対しての理解を深めていただけると幸いです。

# 参考資料

- [JavaScriptのフレームワークについて検討してみよう](https://postd.cc/javascript-framework-fatigue/)
- [一人React.js Advent Calendar 2014](https://qiita.com/advent-calendar/2014/reactjs)
- [今話題のReact.jsはどのようなWebアプリケーションに適しているか？ Introduction To React─ Frontrend Conference](https://html5experts.jp/hokaccha/13301/)
- [社内勉強会でReactとFluxについて喋った](https://devpixiv.hatenablog.com/entry/2015/04/27/170944)
- [What is Flux - いまさら始める、クライアントサイドの設計とFlux](https://www.slideshare.net/axross/what-is-flux)
