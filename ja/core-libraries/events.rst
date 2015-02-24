イベントシステム
################

.. versionadded:: 2.1

メンテナンス性の高いアプリケーションの創造は、科学でもあり芸術でもあります。
良く知られていることですが、高い品質のコードを保持するための鍵は、
オブジェクトが疎結合すると同時に、高い凝集度も合わせ持つということです。
結合が疎であるということが、あるクラスがいかに少ししか外部のオブジェクトに "束縛されて" おらず
どの程度そのクラスがそれらの外部オブジェクトに依存しているかの指標となる一方で、
高い凝集度は、クラスの全てのメソッドおよびプロパティがそのクラス自身と強く関連を持ちつつ
他のオブジェクトがやるべき仕事をしようとはしないということを意味します。

CakePHPのストラクチャとデフォルトのライブラリの多くがあなたのこのゴールの達成を
手助けしてくれるとは言え、依存関係をガチガチにコードすることなくシステム内の
別の箇所とクリーンにやりとりすることが必要な場面も確かにあり、やがて凝集度は失われ、
クラスの結合度は増加してしまいます。ソフトウェア工学においてまさに成功したデザインパターンと言えるのが、
オブジェクトがイベントを発生させることができ、無名のリスナーに対して内部状態の変化について通知する
Observerパターンです。

Observerパターンにおけるリスナーは、そのようなイベントを受信することが可能で、
それらに基づいて振る舞いを選択したり、サブジェクトの状態を変更したり、
単に何かを記録したりします。もしあなたがすでにjavascriptを使っていたなら、
すでにイベント駆動プログラミングに親しんでいることでしょう。

そのオブジェクト指向設計に忠実なまま、CakePHPは、jQueryなどの一般的なJavaScriptフレームワーク
においてイベントがトリガーされ管理される方法のいくつかの側面をエミュレートします。
この実装においてイベントオブジェクトは、情報と、任意の時点でのイベント伝播を停止させる能力を保持したまま、
全てのリスナーに行き渡ります。リスナーは自分自身を登録することが可能であるか、
もしくは他のオブジェクトにそのタスクを委任することができ、まだ実行されていないコールバックのために、
状態とイベント自体を変更する機会を持ちます。

イベントマネージャーとの対話
============================

さて、あなたが買い物カートのプラグインをビルドしていて、だけど、出荷の仕組み、メールの送信、
在庫を減らすこととかをごちゃまぜにするなんてことは全く望んでおらず、
他のプラグインやアプリケーションコードで個別にそれらの事を処理することを望んでいるとしましょう。
いつもの、それこそObserverパターンを使用しない場合、不断の努力でビヘイビアをモデルに
（ひょっとしたらいくつかのコンポーネントをコントローラに）取り付けることによって、
この望みをかなえるでしょう。

そうすることは、外部でビヘイビアをロードするかフックをプラグインコントローラに取り付けるかするために
コードを用意しなくてはならないので、ほとんどの場合ひとつの挑戦を意味します。
CakePHP 2.1 以前の何人かの開発者は、この問題を解決するために汎用イベントシステムを実装することにし、
そのうちいくつかのシステムはプラグインとして提供されていました。今あなたは、組み込みの
イベントマネージャーによって、プラグインとアプリケーションコードの関係をきれいに切り離してくれる、
標準的な多目的イベントシステムから恩恵を受けることができます。

イベントのディスパッチ
----------------------

それでは先ほどの例に戻りましょう。私たちは購買ロジックを管理する `Order` モデルと、
恐らく、注文の詳細を保存したりその他もろもろのロジックを行うための `place` メソッドを持つことになるでしょう::

    // Cart/Model/Order.php
    class Order extends AppModel {

        public function place($order) {
            if ($this->save($order)) {
                $this->Cart->remove($order);
                $this->sendNotificationEmail();
                $this->decrementFromStock();
                $this->updateUserStatistics();
                // ...
                return true;
            }
            return false;
        }
    }

まあ、全然よく見えないですよね。プラグインは電子メールの送信について何らかの仮定を立てるべきではないし、
さらにそれから商品をデクリメントする目録データを持っていない可能性すらあるし、
そして使用状況に関する統計情報のトラッキングを行うにはどう見ても最適の場所ではありません。
つまり、我々には別の解決策が必要と言うわけなので、さて、イベント·マネージャーを用いて
これを書き直してみましょう::

    // Cart/Model/Order.php
    App::uses('CakeEvent', 'Event');
    class Order extends AppModel {

        public function place($order) {
            if ($this->save($order)) {
                $this->Cart->remove($order);
                $this->getEventManager()->dispatch(new CakeEvent('Model.Order.afterPlace', $this, array(
                    'order' => $order
                )));
                return true;
            }
            return false;
        }
    }

イベントクラスとメソッドを導入することで、ずいぶんすっきりしましたね。最初に気づくことは
``getEventManager()`` の呼び出しではないでしょうか。これはデフォルトの状態ですべてのモデル、コントローラ、
ビューで使用可能なメソッドです。このメソッドは、モデル間で同じマネージャーのインスタンスを返すことはなく、
コントローラとモデルの間でも共有されていないにも関わらず、コントローラとビューの間では共有されています。
この実装の詳細をいかにして攻略するかは、後ほど検討します。

``getEventManager`` メソッドは :php:class:`CakeEventManager` のインスタンスを返します。
そしてイベントのタスクを処理するために、 :php:class:`CakeEvent` クラスのインスタンスを受け取る
:php:meth:`CakeEventManager::dispatch()` を使用します。それでは、イベントを処理するプロセスを
詳しく見てみましょう::

    new CakeEvent('Model.Order.afterPlace', $this, array(
        'order' => $order
    ));

:php:class:`CakeEvent` は、そのコンストラクタに3つの引数を受け取ります。最初のものはイベント名で、
読みやすくすると同時にできるだけ唯一性を維持することを心掛けてください。
我々は次のような規則を提案します： レイヤーレベルで発生する一般的なイベントのためには
`Layer.eventName` と言う記述方法 (例. `Controller.startup`, `View.beforeRender`) を、
そして あるレイヤーの特定のクラスで発生するイベントのためには
`Layer.Class.eventName` 例えば `Model.User.afterRegister` や `Controller.Courses.invalidAccess`
と言う記述方法を。

2つ目の引数は`subject`です。サブジェクトとはイベントに関連付けられているオブジェクトを意味し、
通常それ自身に関するイベントをトリガーしているものと同じクラスであり、
`$this` の使用が一般的なケースとなります。とは言え、:php:class:`Component` が
コントローライベントをトリガしたりもできます。サブジェクトクラスは重要です。
なぜなら、リスナーがオブジェクトのプロパティへの即時アクセスを取得し、
それらをその場で検査したり変更するチャンスを持てるようになるからです。

最後に、3番目の引数はイベントのパラメータです。これは、リスナーがそれに基づいて
行動できるようにするための任意のデータです。これは、どのような型の引数でも指定できますが、
検査を容易にするために連想配列を渡すことをお勧めします。

:php:meth:`CakeEventManager::dispatch()` メソッドは、引数としてイベントオブジェクトを受け取り、
すべてのリスナーとコールバックにこのオブジェクトを伝達させながら通知します。このようにして、
リスナーは `afterPlace` イベントにまつわる、その他のすべてのロジックを処理できるようになるので、
あなたは時間をログに取ったり、電子メールを送信したり、ユーザーの統計情報を更新したりを
別々のオブジェクトで行うことができ、必要があればオフラインのタスクにそれを委任することさえできるのです。

コールバックの登録
------------------

新しいafterPlaceイベントにコールバックやオブザーバを登録するにはどうすればよいのでしょうか？
これは多種多様な異なる実装がなされますが、どのような場合であっても新しいアクターを登録する
:php:meth:`CakeEventManager::attach()` メソッドを呼び出す必要はあります。わかりやすくするために、
このプラグインにおいてコントローラでコールバックを使用可能であることを我々は知っており、
このコントローラは、それらを責任を持って接続するとしましょう。可能なコードは次のようになります::

    // Listeners configured somewhere else, maybe a config file:
    Configure::write('Order.afterPlace', array(
        'email-sending' => 'EmailSender::sendBuyEmail',
        'inventory' => array($this->InventoryManager, 'decrement'),
        'logger' => function($event) {
            // Anonymous function are only available in PHP 5.3+
            CakeLog::write('info', 'A new order was placed with id: ' . $event->subject()->id);
        }
    ));

    // Cart/Controller/OrdersController.php
    class OrdersController extends AppController {

        public function finish() {
            foreach (Configure::read('Order.afterPlace') as $l) {
                $this->Order->getEventManager()->attach($l, 'Model.Order.afterPlace');
            }
            if ($this->Order->place($this->Cart->items())) {
                // ...
            }
        }
    }

これはそのための最善な方法ではないかもしれないので、リスナーをオブジェクトのイベントマネージャーに
アタッチするためのあなた自身のやり方を考えてください。 `Configure` クラスを使用してそれらを定義するという
この単純な方法は単に教科書的に書いたにすぎません。この小さな例が私たちに示すことは、
どのようなタイプのコールバックであってもマネージャーにアタッチ可能だということです。
すでにお分かりだと思いますが、この `attach` メソッドはすべての有効なPHPコールバックのタイプ、
つまり文字列で表された static function の呼び出し、クラスインスタンスとメソッドを保持した配列、
もしPHP5.3以上を利用しているなら無名関数、などをうけとります。アタッチされたコールバックは全て、
第1の引数としてイベントオブジェクトを受け取ります。

:php:meth:`CakeEventManager::attach()` は3つの引数を受け取ります。左端の1つはコールバック自身、
PHPが呼び出し可能な関数として扱うことができる何かです。第二引数にはイベント名で、
`CakeEvent` オブジェクトはこれとマッチした名前でディスパッチされたときにのみ動作します。
最後の引数はコールバックのプライオリティ、および渡される引数のプライオリティを設定するためのオプションの配列です。

リスナーの登録
--------------

リスナーは、イベントのためにコールバックを登録する選択肢の一つでありとてもクリーンな方法です。
これは、コールバックをいくつか登録したいとあなたが望む任意のクラスに対し
:php:class:`CakeEventListener` インターフェイスを実装することによって実現されます。
このインターフェイスを実装しているクラスは、``implementedEvents()`` メソッドを提供し、
クラスが処理するすべてのイベント名を持つ連想配列を返す必要があります。

それでは先ほどの例を追いながら、有意義な情報を計算してグローバル・サイトの統計へと
コンパイルする役割を果たすUserStatisticクラスがあると仮定しましょう。
このクラス内のメソッドをトリガーするためには、手製のstaticな関数の実装や他のいかなる回避策よりも、
このクラスのインスタンスをコールバックとして渡すのが自然でしょう。リスナーは次のように作成します::

    App::uses('CakeEventListener', 'Event');
    class UserStatistic implements CakeEventListener {

        public function implementedEvents() {
            return array(
                'Model.Order.afterPlace' => 'updateBuyStatistic',
            );
        }

        public function updateBuyStatistic($event) {
            // Code to update statistics
        }
    }

    // Attach the UserStatistic object to the Order's event manager
    $statistics = new UserStatistic();
    $this->Order->getEventManager()->attach($statistics);

上記のコードを見るとわかるように、`attach` 関数は `CakeEventListener` インターフェイスの
インスタンスを操ることができます。内部的には、イベント·マネージャーは `implementedEvents`
メソッドが返す配列を読み取り、ただちにコールバックを結びつけます。

.. _event-priorities:

プライオリティの設置
--------------------

いくつかのケースでは、コールバックを実行させて他の実行済みのコールバックとの前後関係を
明らかにしたいと思うでしょう。例としてユーザーの統計情報の場合についてもう一度考えて見ましょう。
このメソッドを実行させる意義があるのは、イベントはキャンセルされておらず、エラーもなく、
その他のコールバックが注文の状態そのものを変更させていないことが明らかになった時に限られます。
このような場合、プライオリティを用います。

プライオリティは、コールバック自体に関連付けられている数字を使用して処理されます。
数字が大きいほど、後に実行されるメソッドです。すべてのコールバックとリスナメソッドの
デフォルトの優先度は `10` に設定されています。もしメソッドをもっと早く実行したい場合は、
このデフォルト値よりも小さい任意の値(`1` でもいいし負の値でも動作するでしょう)の使用が
あなたを助けてくれます。逆に、コールバックを他よりもあとに実行させたいなら、
`10` よりも大きい数字を用いてください。

2つのコールバックが同じプライオリティキューに割り当てられるた場合は、
それらはFIFOポリシーで実行され、最初にアタッチされたリスナーメソッドは最初に、
という具合に実行されます。コールバックのプライオリティを設定するためにはattachメソッドを用い、
リスナーのプライオリティを設定するためには `implementedEvent` 関数内での宣言を行います::

    // Setting priority for a callback
    $callback = array($this, 'doSomething');
    $this->getEventManager()->attach($callback, 'Model.Order.afterPlace', array('priority' => 2));

    // Setting priority for a listener
    class UserStatistic implements CakeEventListener {
        public function implementedEvents() {
            return array(
                'Model.Order.afterPlace' => array('callable' => 'updateBuyStatistic', 'priority' => 100),
            );
        }
    }

ご覧のとおり、`CakeEventListener` オブジェクトにおける主な違いは、
コーラブルメソッドとプライオリティを指定するために配列を使用する必要があるということです。
`callable` キーはマネージャーがクラス内のどのような関数が呼ばれるべきかを知るために読み込むであろう、
特別な配列エントリです。

イベントを関数のパラメータとして受け取る
----------------------------------------

一部の開発者は、イベントオブジェクトを受け取ることよりも関数のパラメータとして渡された
イベントのデータを保持する方を好むかもしれません。これはまあ、ちょっと変わった趣味で、
イベントオブジェクトを用いるほうがずっとパワフルと言えますが、以前のイベントシステムとの後方互換性と、
経験豊かな開発者が慣れ親しんだ環境の代替手段とを提供する必要もあったのです。

この方法を選択する場合、プライオリティの設定でやったのと同じように、`attach`メソッドの
3番目の引数に`passParams`オプションを追加するか `implementedEvents` が返す配列に
それを宣言する必要があります::

    // Setting priority for a callback
    $callback = array($this, 'doSomething');
    $this->getEventManager()->attach($callback, 'Model.Order.afterPlace', array('passParams' => true));

    // Setting priority for a listener
    class UserStatistic implements CakeEventListener {
        public function implementedEvents() {
            return array(
                'Model.Order.afterPlace' => array('callable' => 'updateBuyStatistic', 'passParams' => true),
            );
        }

        public function updateBuyStatistic($orderData) {
            // ...
        }
    }

上記のコードでは `doSomething` 関数と `updateBuyStatistic` メソッドは `$event` オブジェクト
の代わりに `$orderData` を受け取ることになります。これは、先ほどの例において、
いくつかのデータを伴って `Model.Order.afterPlace` イベントをトリガするからです::

    $this->getEventManager()->dispatch(new CakeEvent('Model.Order.afterPlace', $this, array(
        'order' => $order
    )));

.. note::
  イベントデータが配列の場合にのみ、このパラメータは関数の引数として渡すことができます。
  他のどのようなデータ型も関数のパラメータに変換することはできません。という訳で、
  ほとんどの場合においてこの選択肢を用いないことが最適な選択となります。

イベントの停止
--------------

イベントを開始した操作がキャンセルされたために、イベントを停止しなくてはならない状況があります。
それ以上処理を進めることが不可能であることをコードが検出した時に保存操作を停止できる、モデルの
コールバック（例えばbeforeSave）において、そのような例を見いだせます。

イベントを停止するためには、コールバックで `false` を返すか、またはイベントオブジェクトで
`stopPropagation` メソッドを呼び出すかのいずれかを行うことができます::

    public function doSomething($event) {
        // ...
        return false; // stops the event
    }

    public function updateBuyStatistic($event) {
        // ...
        $event->stopPropagation();
    }

イベントの停止は2つの異なる効果をもたらせます。最初のものは常に期待することができます:
いかなるのコールバックも停止されて呼び出されることはありません。2番目の結果はオプションで、
イベントをトリガするコードに依存します。例えば `afterPlace` の例では、
すでにデータが保存されカートが空になった後なので、操作をキャンセルするすることには
何の意味もありません。しかしながら、もし `beforePlace` を停止させていたら、イベントは意味を持ちます。

.. To check if an event was stopped, you call the `isStopped()` method in the event object::

イベントが中止されたかどうかを確認するには、イベント·オブジェクト内で `isStopped()` メソッドを呼び出します::

    public function place($order) {
        $event = new CakeEvent('Model.Order.beforePlace', $this, array('order' => $order));
        $this->getEventManager()->dispatch($event);
        if ($event->isStopped()) {
            return false;
        }
        if ($this->Order->save($order)) {
            // ...
        }
        // ...
    }

先の例において、イベントがbeforePlaceの処理の間に停止した場合は、注文内容は保存されません。

結果の取得
----------

コールバックが値を返すたびに、それはイベントオブジェクトの `$result` プロパティに格納されます。
これは、コールバックがメインプロセスのパラメータを更新しようとすることで、
処理の局面を変更する能力を高める幾つかの場面で有用です。再び `beforePlace` を例にとり、
コールバックが $order データを変更することを許可してみましょう。

イベントの結果は、イベントオブジェクトのresultプロパティを直接用いるか、
またはコールバック自体の値を返すことで変更できます::

    // A listener callback
    public function doSomething($event) {
        // ...
        $alteredData = $event->data['order'] + $moreData;
        return $alteredData;
    }

    // Another listener callback
    public function doSomethingElse($event) {
        // ...
        $event->result['order'] = $alteredData;
    }

    // Using the event result
    public function place($order) {
        $event = new CakeEvent('Model.Order.beforePlace', $this, array('order' => $order));
        $this->getEventManager()->dispatch($event);
        if (!empty($event->result['order'])) {
            $order = $event->result['order'];
        }
        if ($this->Order->save($order)) {
            // ...
        }
        // ...
    }

あなたもお気づきかも知れませんが、いかなるイベントのオブジェクトであっても変更可能であり、
この新しいデータが次のコールバックに渡されることは明らかです。 ほとんどの場合、
オブジェクトをイベント·データまたは結果として提供し、そのオブジェクトを直接変更することは、
参照が同一に保たれていて変更がすべてのコールバックの呼び出しを超えて共有できるので、
最適なソリューションです。

コールバック及びリスナーの削除
------------------------------

何らかの理由でイベントマネージャーから任意のコールバックを削除したい場合は、
:php:meth:`CakeEventManager::detach()` を引数の最初の2つの変数を attach のときと
同様の用い方で呼び出すだけで良いです::

    // Attaching a function
    $this->getEventManager()->attach(array($this, 'doSomething'), 'My.event');

    // Detaching the function
    $this->getEventManager()->detach(array($this, 'doSomething'), 'My.event');

    // Attaching an anonymous function (PHP 5.3+ only);
    $myFunction = function($event) { ... };
    $this->getEventManager()->attach($myFunction, 'My.event');

    // Detaching the anonymous function
    $this->getEventManager()->detach($myFunction, 'My.event');

    // Attaching a CakeEventListener
    $listener = new MyCakeEventLister();
    $this->getEventManager()->attach($listener);

    // Detaching a single event key from a listener
    $this->getEventManager()->detach($listener, 'My.event');

    // Detaching all callbacks implemented by a listener
    $this->getEventManager()->detach($listener);

グローバルイベントマネージャー
==============================

前述したように、あるオブジェクト内の任意のイベント·マネージャーにObserverを
アタッチするのが難しいことがあるかも知れません。あるイベントに対して、
それをトリガーするオブジェクトのインスタンスにアクセスすること無しに、
コールバックをアタッチできなくてはならない一定のケースがあります。また、
設定に基づいて、コールバックのマネージャーへのローディングを、
各々異なるメカニズムに実装してしまわないように、CakePHPは
グローバルイベントマネージャーの概念を提供します。

グローバルマネージャーは、app dispatches においてすべてのイベントを受け取る
``CakeEventManager`` クラスのシングルトンインスタンスです。これは強力かつ柔軟ですが、
もし使用するなら、より一層イベントを処理するときに注意を払う必要があります。

もう一度コンセプトを正しく設定するために、`beforePlace` の例を用いながら、
`getEventManager` 関数で返されたローカルのイベントマネージャーを用いていたころの
私達を再び思い出しましょう。内部的には、このローカルイベントマネージャーは、
アタッチされたコールバックをトリガーする前に、イベントをグローバルのイベントマネージャーに
ディスパッチしています。各マネージャーの優先度は独立しており、グローバルなコールバックは
独自のプライオリティキューで発生し、ローカルのコールバックは
それぞれのプライオリティの順序で呼び出されます。

グローバルイベントマネージャーにアクセスするには static 関数を呼び出すように簡単で、
次の例では、`beforePlace` イベントにグローバルイベントをアタッチしています::

    // In any configuration file or piece of code that executes before the event
    App::uses('CakeEventManager', 'Event');
    CakeEventManager::instance()->attach($aCallback, 'Model.Order.beforePlace');

ご覧のとおり、イベントマネージャーインスタンスへのアクセスの取得方法を単に変更するだけで、
イベントのトリガー、アタッチ、デタッチ、ストップなど既に学習したのと同じ概念を適用することができます。

あなたが考慮すべき重要な点は、同名でありながら異なるサブジェクトを伴ってトリガーされる
イベントが存在することがあるので、グローバルマネージャーでアタッチされたコールバックでは
通常、バグを防ぐためにイベントオブジェクト内でそれをチェックすることが求められているということです。
極度な柔軟性は、同時に極度な複雑性をも意味することを覚えておいてください。

すべてのモデルのbeforeFindsを補足したい一方で、もし Cart モデルの場合ならロジックは実行不可という、
こんなコールバックを考えてみてください::

    App::uses('CakeEventManager', 'Event');
    CakeEventManager::instance()->attach('myCallback', 'Model.beforeFind');

    public function myCallback($event) {
        if ($event->subject() instanceof Cart) {
            return;
        }
        return array('conditions' => ...);
    }

最後に
======

イベントシステムはあなたのアプリケーション内の複雑な関係を分離させる偉大な方法であり、
クラスに凝集と疎結合の両者をもたらしますが、それにもかかわらず、
これはすべての問題の解決策というわけではありません。実のところ、
ほとんどのアプリケーションは全くこの機能を必要とせず、ビヘイビアやコンポーネント、
ヘルパーを用いる様な感じでコールバックを実装するすようになってきたときは、
他の選択肢から検討することをおすすめします。

偉大な力には偉大な責任が伴うことを心に留めておいてください。
この方法でクラスを疎結合させるということは、コード上でより多くの
そしてより良い結合テストの実行が要求されるということです。
このツールの乱用はあなたのアプリケーションによりよいアーキテクチャをもたらすなんてことはなく、
全くその逆に、コードの可読性を著しく下げるでしょう。一方それとは対照的に、
本当に必要なものに限って賢くそれを使用するのならば、コードの取り扱い、テスト、
そして統合させることを容易にしてくれることでしょう。

その他の資料
============

.. toctree::
    :maxdepth: 1

    /core-libraries/collections
    /models/behaviors
    /controllers/components
    /views/helpers


.. meta::
    :title lang=ja: Events system
    :keywords lang=ja: events, dispatch, decoupling, cakephp, callbacks, triggers, hooks, php
