Rails アプリケーションを設定する
==============================

本ガイドでは、Railsアプリケーションで利用可能な設定と初期化機能について解説します。

このガイドの内容:

* Railsアプリケーションの動作を調整する方法
* アプリケーション開始時に実行したいコードを追加する方法

--------------------------------------------------------------------------------


初期化コードの置き場所
---------------------------------

Railsには初期化コードの置き場所が4箇所あります。

* `config/application.rb`
* 環境ごとの設定ファイル
* イニシャライザファイル
* アフターイニシャライザファイル

Rails実行前にコードを実行する
-------------------------

アプリケーションでRails自体が読み込まれる前に何らかのコードを実行する必要が生じることがまれにあります。その場合は、実行したいコードを`config/application.rb`ファイルの`require 'rails/all'`行の上に書いてください。

Railsコンポーネントを構成する
----------------------------

一般に、Railsの設定作業には、Rails自身の設定と、Railsのコンポーネントの設定があります。`config/application.rb`および環境固有の設定ファイル（`config/environments/production.rb`など）に設定を記入すると、Railsのすべてのコンポーネントにそれらの設定が反映されます。

たとえば、`config/application.rb`ファイルに以下の設定を追加できます。

```ruby
config.time_zone = 'Central Time (US & Canada)'
```

上はRails自身のための設定ですが、個別のRailsコンポーネントに設定を反映するときにも、以下のように`config/application.rb`内の同じ`config`オブジェクトを利用できます。

```ruby
config.active_record.schema_format = :ruby
```

この設定は、Active Record固有の設定に使われます。

WARNING: 関連付けされたクラスを直接呼び出すのではなく、必ずpublicな設定メソッドを使うこと。例: `ActionMailer::Base.options`ではなく`Rails.application.config.action_mailer.options`を使う。

NOTE: 設定をクラスに直接適用する必要がある場合は、イニシャライザで[`ActiveSupport::LazyLoadHooks`](https://api.rubyonrails.org/classes/ActiveSupport/LazyLoadHooks.html)をお使いください（初期化が完了する前にクラスがオートロードされるのを避けるため）。初期化中にオートロードされるとアプリの再読み込みを安全に繰り返せなくなるため、失敗します。

### Rails全般の設定

Rails全般に対する設定を行うには、`Rails::Railtie`オブジェクトを呼び出すか、`Rails::Engine`や`Rails::Application`のサブクラスを呼び出します。

#### `config.after_initialize`

Railsによるアプリケーションの初期化が完了した**後に**実行されるブロックを渡せます。アプリケーションの初期化には、「フレームワーク自体の初期化」「エンジンの初期化」「`config/initializers`に記述されたすべてのアプリケーション初期化処理の実行」が含まれます。ここで渡すブロックは**rakeタスクで実行される**ことにご注意ください。このブロックは、他のイニシャライザによってセットアップ済みの値を設定するのに便利です。

```ruby
config.after_initialize do
  ActionView::Base.sanitized_allowed_tags.delete 'div'
end
```

#### `config.asset_host`

アセットを置くホストを設定します。この設定は、アセットの置き場所がCDN（Contents Delivery Network）の場合や、別のドメインエイリアスを使うとブラウザの同時実行制限にひっかかるのを避けたい場合に便利です。この設定は`config.action_controller.asset_host`のショートハンドです。

#### `config.autoload_once_paths`

サーバーへのリクエストごとにクリアされない定数を自動読み込みするパスの配列を渡せます。この設定は`config.cache_classes`が`false`（developmentモードのデフォルト値）の場合に関連しています。それ以外の場合、自動読み込みは1度しか行われません。この配列内にあるすべての要素は`autoload_paths`に存在しなければなりません。デフォルト値は空の配列です。

#### `config.autoload_paths`

Railsが定数を自動読み込みするパスの配列を渡せます。`config.autoload_paths`のデフォルト値は、`app`以下のすべてのディレクトリです。[Rails 6](upgrading_ruby_on_rails.html#オートローディング)以降は、この設定の変更は推奨されません。詳しくは[定数の自動読み込みと再読み込み](autoloading_and_reloading_constants.html)を参照してください。

#### `config.add_autoload_paths_to_load_path`

`$LOAD_PATH`に自動読み込みのパスを追加すべきかどうかを指定します。このフラグはデフォルトで`true`ですが、`:zeitwerk`モードでは早い段階で`config/application.rb`で`false`に設定することをおすすめします。Zeitwerkは内部で絶対パスが使われ、`:zeitwerk`モードで動作するアプリケーションでは`require_dependency`が不要なので、モデルやコントローラやジョブなどが`$LOAD_PATH`に存在する必要はありません。これを`false`に設定することで、Rubyが`require`呼び出しを相対パスで解決するときにディレクトリのチェックが不要になり、インデックスの構築も不要になるので、Bootsnapの動作やメモリを節約できます。

#### `config.cache_classes`

アプリケーションのクラスやモジュールをリクエストごとに再読み込みするか（=キャッシュしないかどうか）どうかを指定します。`config.cache_classes`のデフォルト値は、developmentモードでは`false`なのでコードの更新がすぐ反映され、productionモードの場合は`true`なので高速に動作します。testモードでは、spring gemがインストールされている場合はデフォルトで`false`、そうでない場合は`true`になります。

#### `config.beginning_of_week`

アプリケーションにおける週の初日を設定します。引数には、曜日を表す有効なシンボルを渡します(`:monday`など)。

#### `config.cache_store`

Railsでのキャッシュ処理に使われるキャッシュストアを設定します。指定できるオプションは次のシンボル`:memory_store`、`:file_store`、`:mem_cache_store`、`:null_store`、`:redis_cache_store`のいずれか、またはキャッシュAPIを実装するオブジェクトです。デフォルト値は`:file_store`です。ストアごとの設定オプションについては[キャッシュストア](caching_with_rails.html#キャッシュストア)を参照してください。

#### `config.colorize_logging`

出力するログ情報にANSI色情報を与えるかどうかを指定します。デフォルト値は`true`です。

#### `config.consider_all_requests_local`

このフラグが`true`の場合、エラー発生の種類を問わず詳細なデバッグ情報をHTTPレスポンスに出力し、`Rails::Info`コントローラがアプリケーションの実行時コンテキストを`/rails/info/properties`に出力します。このフラグはdevelopment環境とtest環境では`true`、production環境では`false`に設定されます。より細かく制御したい場合は、このフラグを`false`に設定してから、コントローラで`show_detailed_exceptions?`メソッドを実装し、エラー時にデバッグ情報を出力したいリクエストをそこで指定します。

#### `config.console`

これを用いて、コンソールで`rails console`を実行する時に使われるクラスをカスタマイズできます。このメソッドは`console`ブロックで使うのが最適です。

```ruby
console do
  # このブロックはコンソールで実行されるときしか呼び出されない
  # 従ってpryを安全にrequireできる
  require "pry"
  config.console = Pry
end
```

#### `config.disable_sandbox`

コンソールをsandboxモードで起動してよいかどうかを制御します。これは、sandboxコンソールのセッションを長時間動かしっぱなしにするとデータベースサーバーのメモリが枯渇するのを避けるうえで有用です。デフォルト値は`false`です。

#### `config.eager_load`

`true`にすると、登録された`config.eager_load_namespaces`をeager loadingします。ここにはアプリケーション、エンジン、Railsフレームワークを含むあらゆる登録済み名前空間が含まれます。

#### `config.eager_load_namespaces`

ここに登録した名前は、`config.eager_load`が`true`のときにeager loadingされます。登録された名前空間は、必ず`eager_load!`メソッドに応答しなければなりません。

#### `config.eager_load_paths`

パスの配列を引数に取ります。起動時のRailsは、cache_classesがオンの場合にこのパスからeager loadingします。デフォルトではアプリケーションの`app/`ディレクトリ以下のすべてのディレクトリが対象です。

#### `config.enable_dependency_loading`

`true`の場合、`config.cache_classes`が`true`に設定されていてもアプリケーション起動時の自動読み込みを有効にします。デフォルト値は`false`です。

#### `config.encoding`

アプリケーション全体のエンコーディングを指定します。デフォルト値はUTF-8です。

#### `config.exceptions_app`

例外が発生したときに`ShowException`ミドルウェアによって呼び出される例外アプリケーションを設定します。デフォルト値は`ActionDispatch::PublicExceptions.new(Rails.public_path)`です。

#### `config.debug_exception_response_format`

developmentモードで発生したエラーのレスポンスで用いられるフォーマットを設定します。通常のアプリケーションの場合は`:default`が、APIのみの場合は`:api`がデフォルトで設定されます。

#### `config.file_watcher`

`config.reload_classes_only_on_change`が`true`の場合に、ファイルシステム上のファイル更新検出に使われるクラスを指定します。デフォルトのRailsでは`ActiveSupport::FileUpdateChecker`、および`ActiveSupport::EventedFileUpdateChecker`（これは[listen](https://github.com/guard/listen)に依存します）が指定されます。カスタムクラスはこの`ActiveSupport::FileUpdateChecker` APIに従わなければなりません。

#### `config.filter_parameters`

パスワードやクレジットカード番号など、ログに出力したくないパラメータをフィルタで除外するのに用います。また、Active Recordオブジェクトに対して`#inspect`を呼び出した際に、データベースの機密性の高い値をフィルタで除外します。デフォルトのRailsでは`config/initializers/filter_parameter_logging.rb`に以下の記述を追加することでパスワードをフィルタで除外しています。

```ruby
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt, :certificate, :otp, :ssn
]
```

パラメータのフィルタは正規表現の**部分一致**によって行われます（訳注: 他のパラメータ名が誤って部分一致しないようご注意ください）。

#### `config.force_ssl`

すべてのリクエストをHTTPSプロトコル下で実行するよう強制し、URL生成でも"https://"をデフォルトのプロトコルに設定します。HTTPSの強制は`ActionDispatch::SSL`ミドルウェアによって行われ、`config.ssl_options`で設定できます。詳しくはAPIドキュメント[`ActionDispatch::SSL`](https://api.rubyonrails.org/classes/ActionDispatch/SSL.html)を参照してください。

#### `config.javascript_path`

アプリのJavaScriptを保存するパスを、`app/`ディレクトリからの相対パスで設定します。デフォルト値は`javascript`です（[webpacker](https://github.com/rails/webpacker)で使われます）。アプリで設定済みの`javascript_path`は`autoload_paths`から除外されます。

#### `config.log_formatter`

Railsロガーのフォーマットを定義します。このオプションは、デフォルトではすべてのモードで`ActiveSupport::Logger::SimpleFormatter`のインスタンスを使います。`config.logger`を設定する場合は、この設定が`ActiveSupport::TaggedLogging`インスタンスでラップされるより前の段階で、フォーマッターの値を手動で渡さなければなりません（Railsはこの処理を自動では行いません）。

#### `config.log_level`

Railsのログ出力をどのぐらい詳細にするかを指定します。デフォルト値は、production環境では`:info`、それ以外の環境では`:debug`です。指定可能な出力レベルは`:debug`、`:info`、`:warn`、`:error`、`:fatal`、`:unknown`です。

#### `config.log_tags`

「`request`オブジェクトが応答するメソッド」「`request`オブジェクトを受け取る`Proc`」または「`to_s`に応答できるオブジェクト」のリストを引数に取ります。これは、ログの行にデバッグ情報をタグ付けする場合に便利です。たとえばサブドメインやリクエストidを指定可能で、これらはマルチユーザーのproductionアプリケーションのデバッグで非常に有用です。

#### `config.logger`

`Rails.logger`で使われるロガーやRails関連のあらゆるロガー（`ActiveRecord::Base.logger`など）を指定します。デフォルトでは、`ActiveSupport::Logger`のインスタンスをラップする`ActiveSupport::TaggedLogging`のインスタンスが指定されます。なお`ActiveSupport::Logger`はログを`log/`ディレクトリに出力します。ここにカスタムロガーを指定できますが、互換性を完全にするには以下のガイドラインに従わなければなりません。

* フォーマッターをサポートする場合は、`config.log_formatter`の値を手動でロガーに代入しなければなりません。
* タグ付きログをサポートする場合は、そのログのインスタンスを`ActiveSupport::TaggedLogging`でラップしなければなりません。
* ログ出力の抑制をサポートするには、`LoggerSilence`モジュールを`include`しなければなりません。`ActiveSupport::Logger`クラスは既にこれらのモジュールに`include`されています。

```ruby
class MyLogger < ::Logger
  include ActiveSupport::LoggerSilence
end

mylogger           = MyLogger.new(STDOUT)
mylogger.formatter = config.log_formatter
config.logger      = ActiveSupport::TaggedLogging.new(mylogger)
```

#### `config.middleware`

アプリケーションで使うミドルウェアをカスタマイズできます。詳細については[ミドルウェアを設定する](#ミドルウェアを設定する)の節を参照してください。

#### `config.rake_eager_load`

`true`にすると、Rakeタスク実行中にアプリケーションをeager loadingします。デフォルト値は`false`です。

#### `config.reload_classes_only_on_change`

監視しているファイルが変更された場合にのみクラスを再読み込みするかどうかを指定します。デフォルトでは、`autoload_path`で指定されたすべてのファイルが監視対象となり、デフォルトで`true`が設定されます。`config.cache_classes`が`true`の場合、このオプションは無視されます。

#### `config.credentials.content_path`

暗号化済みcredentialの探索パスを設定します。

#### `config.credentials.key_path`

暗号化キーの探索パスを設定します。

#### `secret_key_base`

このメソッドは、改ざん防止のためにアプリケーションのセッションを既知の秘密キーと照合するキーを指定するときに使います。test環境とdevelopment環境の場合はランダムに生成されたキーを使います。その他の環境ではキーを`config/credentials.yml.enc`に設定すべきです。

#### `config.require_master_key`

`ENV["RAILS_MASTER_KEY"]`環境変数または`config/master.key`ファイルでマスターキーを取得できない場合はアプリを起動しないようにします。

#### `config.public_file_server.enabled`

`public/`ディレクトリ内の静的アセットを配信するかどうかを指定します。デフォルトでは`true`が設定されますが、production環境ではアプリケーションを実行するNginxやApacheなどのサーバーが静的アセットを扱う必要があるので、`false`に設定されます。デフォルトの設定とは異なり、WEBrickを使うアプリケーションをproductionモードで実行したり（WEBrickをproductionで使うことは推奨されません）テストしたりする場合は`true`に設定します。そうしないとページキャッシュが利用できなくなり、`public/`ディレクトリ以下に常駐する静的ファイルへのリクエストも有効になりません。

#### `config.session_store`

セッションの保存に使うクラスを指定します。指定できる値は`:cookie_store`（デフォルト）、`:mem_cache_store`、`:disabled`です。`:disabled`を指定すると、Railsでセッションが扱われなくなります。デフォルトでは、アプリケーション名と同じ名前のcookieストアがセッションキーとして使われます。カスタムセッションストアを指定することも可能です。

```ruby
config.session_store :my_custom_store
```

カスタムストアは`ActionDispatch::Session::MyCustomStore`として定義しなければなりません。

#### `config.time_zone`

アプリケーションのデフォルトタイムゾーンを設定し、Active Recordで認識できるようにします。

### アセットを設定する

#### `config.assets.enabled`

アセットパイプラインを有効にするかどうかを指定します。デフォルト値は`true`です。

#### `config.assets.css_compressor`

CSSの圧縮に用いるプログラムを定義します。このオプションは、`sass-rails`によってデフォルトで設定されます。現時点で他に設定できるのは`:yui`オプションだけです。この場合`yui-compressor` gemを利用します。

#### `config.assets.js_compressor`

JavaScriptの圧縮に使うプログラムを定義します。指定できる値は `:terser`、`:closure`、`:uglifier`、`:yui`です。それぞれ `:terser` gem、`closure-compiler` gem、`uglifier` gem、`yui-compressor` gemに対応します。

#### `config.assets.gzip`

gzipされていないバージョンの作成に加えて、コンパイル済みアセットのgzipバージョン作成も有効にするかどうかを指定するフラグです。デフォルト値は`true`です。

#### `config.assets.paths`

アセット探索用のパスを指定します。この設定オプションにパスを追加すると、アセットの探索先として追加されます。

#### `config.assets.precompile`

`application.css`と`application.js`以外に追加したいアセットがある場合に指定します。これらは`bin/rails assets:precompile`を実行するときに一緒にプリコンパイルされます。

#### `config.assets.unknown_asset_fallback`

アセットがパイプラインにない場合のアセットパイプラインの挙動の変更に使います（sprockets-rails 3.2.0以降を使う場合）。デフォルト値は`true`です。

#### `config.assets.prefix`

アセットを置くディレクトリを指定します。デフォルト値は`/assets`です。

#### `config.assets.manifest`

アセットプリコンパイラのマニフェストファイルで使うフルパスを定義します。デフォルトでは、`config.assets.prefix`で指定された`public/`フォルダ内にある`manifest-<ランダム>.json`という名前のファイルになります。

#### `config.assets.digest`

アセット名に使うSHA256フィンガープリントを有効にするかどうかを指定します。デフォルトで`true`に設定されます。

#### `config.assets.debug`

デバッグ用にアセットの結合と圧縮をやめるかどうかを指定します。`development.rb`ではデフォルトで`true`に設定されます。

#### `config.assets.version`

SHA256ハッシュ生成に使われるオプション文字列です。この値を変更すると、すべてのアセットファイルが強制的に再コンパイルされます。

#### `config.assets.compile`

production環境での動的なSprocketsコンパイルをオンにするかどうかを指定するboolean値です。

#### `config.assets.logger`

ロガーを引数に取ります。このロガーは、Log4rのインターフェイスか、Rubyの`Logger`クラスに従います。デフォルトでは、`config.logger`と同じ設定が使われます。`config.assets.logger`を`false`に設定すると、配信されたアセットのログ出力がオフになります

#### `config.assets.quiet`

アセットへのリクエストのログ出力を無効にします。デフォルトでは`development.rb`で`true`に設定されます。

### ジェネレータを設定する

`config.generators`メソッドを使って、Railsで使うジェネレータを変更できます。このメソッドはブロックを1つ受け取ります。

```ruby
config.generators do |g|
  g.orm :active_record
  g.test_framework :test_unit
end
```

ブロックで利用可能なメソッドの完全なリストは以下のとおりです。

* `force_plural`: モデル名を複数形にするかどうかを指定します。デフォルト値は`false`です。

* `helper`: ヘルパーを生成するかどうかを指定します。デフォルト値は`true`です。

* `integration_tool`: 結合テストの生成に使う統合ツールを定義します。デフォルト値は`:test_unit`です。

* `system_tests`: システムテスト生成に用いる統合ツールを定義します。デフォルト値は`:test_unit`です。

* `orm`: 使うORM (オブジェクトリレーショナルマッピング) を指定します。デフォルト値は`false`であり、この場合はActive Recordが使われます。

* `resource_controller`: `rails generate resource`の実行時にどのジェネレータでコントローラを生成するかを指定します。デフォルト値は`:controller`です。

* `resource_route`: リソースのルーティング定義を生成すべきかどうかを定義します。デフォルト値は`true`です。

* `scaffold_controller`: `resource_controller`と同じではありません。`bin/rails generate scaffold`を実行したときに **scaffold**でどのジェネレータでコントローラを生成するかを指定します。デフォルト値は`:scaffold_controller`です。

* `test_framework`: 利用するテストフレームワークを指定します。デフォルト値は`false`であり、この場合minitestが使われます。

* `template_engine`: ビューのテンプレートエンジン（ERBやHamlなど）を指定します。デフォルト値は`:erb`です。

### ミドルウェアを設定する

どのRailsアプリケーションの背後にも、いくつかの標準的なミドルウェアが配置されています。development環境では、以下の順序でミドルウェアを使います。

#### `ActionDispatch::HostAuthorization`

DNSリバインディングやその他の`Host`ヘッダー攻撃を防ぎます。
development環境ではデフォルトで以下の設定が含まれます。

```ruby
Rails.application.config.hosts = [
  IPAddr.new("0.0.0.0/0"),        # すべてのIPv4アドレス
  IPAddr.new("::/0"),             # すべてのIPv6アドレス
  "localhost",                    # localhost予約済みドメイン
  ENV["RAILS_DEVELOPMENT_HOSTS"]  # 開発用の追加ホストリスト（カンマ区切り）
]
```

development以外の環境では、`Rails.application.config.hosts`は空になり、`Host`ヘッダーチェックは行われません。production環境でヘッダー攻撃から保護したい場合は、以下のように手動でホストを許可する必要があります。

```ruby
Rails.application.config.hosts << "product.com"
```

リクエストのホストは、case演算子（`#===`）で`hosts`のエントリと照合されるので、`Regexp`型、`Proc`型、`IPAddr`型のエントリが`hosts`でサポートされます。以下は正規表現を使った例です。

```ruby
# `www.product.com`や`beta1.product.com`のようなサブドメインからの
# リクエストを許可する
Rails.application.config.hosts << /.*\.product\.com/
```

指定した正規表現はアンカー（`\A`と`\z`）で囲まれるので、ホスト名全体とマッチしなければなりません。たとえば`/product.com/`はアンカーで囲まれると`www.product.com`とのマッチに失敗します。

特殊なケースとして、すべてのサブドメインの許可がサポートされます。

```ruby
# `www.product.com`や`beta1.product.com`のようなサブドメインからの
# リクエストを許可する
Rails.application.config.hosts << ".product.com"
```

Host Authorizationチェックで特定のリクエストを除外するには`config.host_authorization.exclude`を設定します。

```ruby
# /healthcheck/パスへのリクエストをホストチェックから除外する
Rails.application.config.host_authorization = {
  exclude: ->(request) { request.path =~ /healthcheck/ }
}
```

許可されていないホストからのリクエストを受け取ると、デフォルトのRackアプリケーションが`403 Forbidden`レスポンスを返します。この動作は以下のように`config.host_authorization.response_app`を設定することでカスタマイズできます。

```ruby
Rails.application.config.host_authorization = {
  response_app: -> env do
    [400, { "Content-Type" => "text/plain" }, ["Bad Request"]]
  end
}
```

#### `ActionDispatch::SSL`

すべてのリクエストでHTTPSプロトコルを強制します。これは`config.force_ssl`を`true`にすると有効になります。渡すオプションは`config.ssl_options`で設定できます。

#### `ActionDispatch::Static`

静的アセットの配信に使います。`config.public_file_server.enabled`が`false`の場合は無効に設定されます。静的ディレクトリのインデックスファイルが`index`でない場合には、`config.public_file_server.index_name`を設定してください。たとえば、ディレクトリへのリクエストを`index.html`ではなく`main.html`で扱うには、`config.public_file_server.index_name`を`"main"`に設定します。

#### `ActionDispatch::Executor`

スレッドセーフなコード再読み込みを許可します。これは`config.allow_concurrency`が`false`の場合に無効になり、`Rack::Lock`が読み込まれるようになります。`Rack::Lock`はアプリケーションをミューテックスにラップするので、同時に1つのスレッドでしか呼び出されなくなります。

#### `ActiveSupport::Cache::Strategy::LocalCache`

基本的なメモリバックアップ式キャッシュとして機能します。このキャッシュはスレッドセーフではなく、単一スレッド用の一時メモリキャッシュとして機能するためのものである点にご注意ください。

#### `Rack::Runtime`

`X-Runtime`ヘッダーを設定します。このヘッダーには、リクエストの実行に要した時間（秒）が含まれます。

#### `Rails::Rack::Logger`

リクエストが開始されたことをログに通知します。リクエストが完了すると、すべてのログをフラッシュします。

#### `ActionDispatch::ShowExceptions`

アプリケーションから返されるすべての例外をrescueし、リクエストがローカルであるか`config.consider_all_requests_local`が`true`に設定されている場合に適切な例外ページを出力します。`config.action_dispatch.show_exceptions`が`false`に設定されていると、常に例外が出力されます。

#### `ActionDispatch::RequestId`

レスポンスで利用できる独自の`X-Request-Id`ヘッダーを作成し、`ActionDispatch::Request#uuid`メソッドを有効にします。`config.action_dispatch.request_id_header`で設定可能です。

#### `ActionDispatch::RemoteIp`

IPスプーフィング攻撃が行われていないかどうかをチェックし、リクエストヘッダーから正しい`client_ip`を取得します。この設定は`config.action_dispatch.ip_spoofing_check`オプションと`config.action_dispatch.trusted_proxies`オプションで変更可能です。

#### `Rack::Sendfile`

bodyがファイルから配信されているレスポンスをインターセプトし、サーバー固有の`X-Sendfile`ヘッダーに差し替えてから送信します。この動作は`config.action_dispatch.x_sendfile_header`で設定可能です。

#### `ActionDispatch::Callbacks`

リクエストを処理する前に、事前コールバックを実行します。

#### `ActionDispatch::Cookies`

リクエストにcookieを設定します。

#### `ActionDispatch::Session::CookieStore`

セッションをcookieに保存する役割を担います。`config.action_controller.session_store`の値を変更すると別のミドルウェアを使えます。これに渡されるオプションは`config.action_controller.session_options`で設定できます。

#### `ActionDispatch::Flash`

`flash`キーを設定します。これは、`config.action_controller.session_store`に値が設定されている場合にのみ有効です。

#### `Rack::MethodOverride`

`params[:_method]`が設定されている場合にHTTPメソッドの上書きを許可します。これは、HTTPでPATCH、PUT、DELETEメソッドを使えるようにするミドルウェアです。

#### `Rack::Head`

HEADリクエストをGETリクエストに変換して配信します。

#### カスタムミドルウェアを追加する

`config.middleware.use`メソッドを使うと、上記以外に独自のミドルウェアを追加することもできます。

```ruby
config.middleware.use Magical::Unicorns
```

上の指定により、`Magical::Unicorns`ミドルウェアがスタックの最後に追加されます。あるミドルウェアの前に別のミドルウェアを追加したい場合は`insert_before`を使います。

```ruby
config.middleware.insert_before Rack::Head, Magical::Unicorns
```

ミドルウェアはインデックスを用いて挿入箇所に正確に指定できます。たとえば、`Magical::Unicorns`ミドルウェアをスタックの最上位に挿入するには次のように設定します。

```ruby
config.middleware.insert_before 0, Magical::Unicorns
```

あるミドルウェアの後に別のミドルウェアを追加したい場合は`insert_after`を使います。

```ruby
config.middleware.insert_after Rack::Head, Magical::Unicorns
```

これらのミドルウェアは、まったく別のものに差し替えることもできます。

```ruby
config.middleware.swap ActionController::Failsafe, Lifo::Failsafe
```

ミドルウェアをある位置から別の位置に移動できます。

```ruby
config.middleware.move_before ActionDispatch::Flash, Magical::Unicorns
```

上は`Magical::Unicorns`ミドルウェアを`ActionDispatch::Flash`の直前に移動します。

以下のように直後に移動することもできます。

```ruby
config.middleware.move_after ActionDispatch::Flash, Magical::Unicorns
```

ミドルウェアをスタックから完全に取り除くこともできます。

```ruby
config.middleware.delete Rack::MethodOverride
```

### i18nを設定する

以下のオプションはすべて`i18n`（internationalization: 国際化）ライブラリ用のオプションです。

#### `config.i18n.available_locales`

アプリケーションで利用できるロケールを許可リスト化します。デフォルトでは、ロケールファイルにあるロケールキーはすべて有効になりますが、新しいアプリケーションの場合、通常は`:en`だけです。

#### `config.i18n.default_locale`

アプリケーションのi18nで使われるデフォルトのロケールを設定します。デフォルト値は`:en`です。

#### `config.i18n.enforce_available_locales`

これをオンにすると、`available_locales`リストで宣言されていないロケールはi18nに渡せなくなります。利用できないロケールがある場合は`i18n::InvalidLocale`例外が発生します。デフォルト値は`true`です。このオプションは、ユーザー入力のロケールが不正である場合のセキュリティ対策であるため、特別な理由がない限り無効にしないことをおすすめします。

#### `config.i18n.load_path`

ロケールファイルの探索パスを設定します。デフォルト値は`config/locales/*.{yml,rb}`です。

#### `config.i18n.raise_on_missing_translations`

コントローラやビューで訳文が見つからない場合にエラーをraiseするかどうかを指定します。デフォルト値は`false`です。

#### `config.i18n.fallbacks`

訳文がない場合のフォールバック動作を設定します。ここではオプションの3つの使い方を説明します。

  * デフォルトのロケールをフォールバック先として使う場合は次のように`true`を設定します。

  ```ruby
  config.i18n.fallbacks = true
  ```

  * ロケールの配列をフォールバック先に使う場合は次のようにします。

  ```ruby
  config.i18n.fallbacks = [:tr, :en]
  ```

  * ロケールごとに個別のフォールバックを設定することも可能です。たとえば`:az`に`:tr`を、`:da`に`:de`と`:en`をそれぞれフォールバック先として指定する場合は、次のようにします。

  ```ruby
  config.i18n.fallbacks = { az: :tr, da: [:de, :en] }
  # または
  config.i18n.fallbacks.map = { az: :tr, da: [:de, :en] }
  ```

### Active Modelを設定する

#### `config.active_model.i18n_customize_full_message`

`full_message`エラーフォーマットを属性レベルやモデルレベルでロケールファイルで上書きしてよいかどうかを制御するboolean値です。デフォルト値は`false`です。

### Active Recordを設定する

`config.active_record`には多くのオプションが含まれています。

#### `config.active_record.logger`

Log4rのインターフェイスまたはデフォルトのRuby Loggerクラスに従うロガーを引数として取ります。このロガーは以後作成されるすべての新しいデータベース接続に渡されます。Active Recordのモデルクラスまたはモデルインスタンスに対して`logger`メソッドを呼び出すと、このロガーを取り出せます。ログ出力を無効にするには`nil`を設定します。

#### `config.active_record.primary_key_prefix_type`

主キーカラムの命名法を変更するのに使います。Railsでは、主キーカラムの名前にデフォルトで`id`が使われます (なお`id`にする場合は値の設定は不要です)。`id`以外に以下の2つを指定できます。

* `:table_name`: たとえばCustomerクラスの主キーは`customerid`になります
* `:table_name_with_underscore`: たとえばCustomerクラスの主キーは`customer_id`になります

#### `config.active_record.table_name_prefix`

テーブル名の冒頭にグローバルに追加したい文字列を指定します。たとえば`northwest_`を指定すると、Customerクラスは`northwest_customers`をテーブルとして探します。デフォルト値は空文字列です。

#### `config.active_record.table_name_suffix`

テーブル名の末尾にグローバルに追加したい文字列を指定します。たとえば`_northwest`を指定すると、Customerクラスは`customers_northwest`をテーブルとして探します。デフォルト値は空文字列です。

#### `config.active_record.schema_migrations_table_name`

スキーママイグレーションのテーブル名に使う文字列を指定します。

#### `config.active_record.internal_metadata_table_name`

内部のメタテーブル名に使う文字列を設定できます。

#### `config.active_record.protected_environments`

破壊的操作を禁止すべき環境名を配列で設定できます。

#### `config.active_record.pluralize_table_names`

Railsが探すデータベースのテーブル名を単数形にするか複数形にするかを指定します。`true`に設定すると、Customerクラスが使うテーブル名は複数形の`customers`になります（デフォルト）。`false`に設定すると、Customerクラスが使うテーブル名は単数形の`customer`になります。

#### `config.active_record.default_timezone`

データベースから日付・時刻を取り出した際のタイムゾーンを`Time.local`（`:local`を指定した場合）と`Time.utc`（`:utc`を指定した場合）のどちらにするかを指定します。デフォルト値は`:utc`です。

#### `config.active_record.schema_format`

データベーススキーマをファイルに書き出すときのフォーマットを指定します。デフォルト値は`:ruby`で、データベースには依存せず、マイグレーションに依存します。`:sql`を指定するとSQL文で書き出されますが、この場合潜在的にデータベースに依存する可能性があります。

#### `config.active_record.error_on_ignored_order`

バッチクエリの実行中にクエリの順序が無視された場合にエラーをraiseすべきかどうかを指定します。オプションは`true`（エラーをraise）または`false`（警告）で、デフォルト値は`false`です。

#### `config.active_record.timestamped_migrations`

マイグレーションファイル名にシリアル番号とタイムスタンプのどちらを与えるかを指定します。デフォルト値は`true`で、タイムスタンプが使われます。複数の開発者が作業する場合は、タイムスタンプの利用をおすすめします。

#### `config.active_record.lock_optimistically`

Active Recordで楽観的ロック（optimistic locking）を使うかどうかを指定します。デフォルト値は`true`（利用する）です。

#### `config.active_record.cache_timestamp_format`

キャッシュキーに含まれるタイムスタンプ値の形式を指定します。デフォルト値は`:usec`です。

#### `config.active_record.record_timestamps`

モデルで発生する`create`操作や`update`操作にタイムスタンプを付けるかどうかを指定します。デフォルト値は`true`です。

#### `config.active_record.partial_inserts`

新規レコード作成で部分書き込みを行うかどうか（挿入時にデフォルトと異なる属性だけを設定するかどうか）を指定するboolean値です。

#### `config.active_record.partial_updates`

既存レコードの更新で部分書き込みを行なうかどうか（「dirty」とマークされた属性だけを更新するか）を指定するboolian値です。データベースで部分書き込みを使う場合は、`config.active_record.lock_optimistically`で楽観的ロックも有効にする必要がある点にご注意ください。これは更新処理が並行して実行された場合に、読み込み中の古い情報に基づいて属性に書き込まれる可能性があるためです。デフォルト値は`true`です。

#### `config.active_record.maintain_test_schema`

テスト実行時にActive Recordがテスト用データベーススキーマを`db/schema.rb`（または`db/structure.sql`）に基づいて最新の状態にするかどうかを指定します。デフォルト値は`true`です。

#### `config.active_record.dump_schema_after_migration`

マイグレーション実行時にスキーマダンプ（`db/schema.rb`または`db/structure.sql`）を行なうかどうかを指定します。このオプションは、Railsが生成する`config/environments/production.rb`では`false`に設定されます。この設定が無指定の場合は、デフォルトの`true`が指定されます。

#### `config.active_record.dump_schemas`

`db:structure:dump`の呼び出し時にデータベーススキーマをダンプするかどうかを指定します。利用可能なオプションは、`:schema_search_path`（デフォルト、`schema_search_path`内のすべてのスキーマをダンプ）、`:all`（`schema_search_path`と無関係にすべてのスキーマをダンプ）、またはスキーマ文字列（カンマ区切り）です。

#### `config.active_record.belongs_to_required_by_default`

`belongs_to`関連付けが存在しない場合にレコードのバリデーションを失敗させるかどうかを指定するboolean値です。

#### `config.active_record.action_on_strict_loading_violation`

関連付けに`strict_loading`が設定されている場合に、例外をraiseするかログに出力するかを設定します。デフォルト値はすべての環境で`:raise`です。これを`:log`に変更すると、例外をraiseせずにロガーに送信できます。

#### `config.active_record.strict_loading_by_default`

`strict_loading`モードをデフォルトで有効にするか無効にするかを指定するboolean値です。デフォルト値は`false`です。

#### `config.active_record.warn_on_records_fetched_greater_than`

クエリ結果のサイズに応じて警告を出す場合の閾値（Threshold）を設定します。あるクエリから返されるレコード数がこの閾値を超えると、警告がログに出力されます。これは、メモリ肥大化の原因となっている可能性のあるクエリを特定するのに利用できます。

#### `config.active_record.index_nested_attribute_errors`

ネストした`has_many`関連付けのエラーをインデックス付きでエラー表示するかどうかを指定します。デフォルト値は`false`です。

#### `config.active_record.use_schema_cache_dump`

（`bin/rails db:schema:cache:dump`で生成された）`db/schema_cache.yml`のスキーマ情報を、データベースにクエリを送信しなくてもユーザーが取得できるようにするかどうかを指定します。デフォルト値は`true`です。

#### `config.active_record.cache_versioning`

キャッシュバージョン変更を伴う安定した`#cache_key`メソッドを`#cache_version`メソッドで使うかどうかを指定します。

#### `config.active_record.collection_cache_versioning`

`ActiveRecord::Relation`型でキャッシュされたオブジェクトが、そのリレーションのキャッシュキーの揮発性の情報（updated atとcountの最大値）をキャッシュキーの再利用サポートのためにキャッシュバージョンに移動したときに、同じキャッシュキーが再利用されるようにします。

#### `config.active_record.has_many_inversing`

`belongs_to`関連付けを`has_many`関連付けにトラバースするときに`inverse_of`のレコードも設定されるようにします。

#### `config.active_record.automatic_scope_inversing`

スコープ付き関連付けで`inverse_of`を自動的に推論するようにします。

#### `config.active_record.legacy_connection_handling`

新しいコネクションハンドリングAPIを有効化できます。この新しいAPIは、マルチプルデータベースを使うアプリケーション向けに粒度の細かいコネクションスワップをサポートします。

#### `config.active_record.destroy_association_async_job`

関連付けられたレコードの非同期削除に使うジョブを指定します。デフォルト値は`ActiveRecord::DestroyAssociationAsyncJob`です。

#### `config.active_record.queues.destroy`

非同期の削除ジョブに使うActive Jobキューを指定できます。このオプションを`nil`にすると、purgeジョブがデフォルトのActive Jobキューに送信されます（`config.active_job.default_queue_name`を参照）。デフォルト値は`nil`です。

#### `config.active_record.enumerate_columns_in_select_statements`

`true`にすると、`SELECT`文に常にカラム名が含まれるようになり、`SELECT * FROM ...`のようなワイルドカードクエリを回避します。これにより、PostgreSQLデータベースにカラムを追加するときなどに、prepared statementのキャッシュエラーを回避できるようになります。デフォルト値は`false`です。

#### `config.active_record.verify_foreign_keys_for_fixtures`

フィクスチャがテストに読み込まれた後で、すべての外部キー制約が有効になるようにします（PostgreSQLとSQLiteのみ）。デフォルト値は`false`です。

#### `config.active_record.query_log_tags_enabled`

クエリコメントをアダプタレベルで有効にするかどうかを指定します。デフォルト値は`false`です。

#### `config.active_record.query_log_tags`

SQLコメントに挿入するキーバリュータグを指定する`Array`を定義します。デフォルト値は、アプリケーション名を返す定義済みのタグ`[ :application ]`です。

#### `config.active_record.cache_query_log_tags`

クエリログタグのキャッシュを有効にするかどうかを指定します。クエリ数が非常に多いアプリケーションでは、クエリログタグのキャッシュを有効にすると、リクエストやジョブの実行中にコンテキストが変更されない場合にパフォーマンスが向上します。デフォルト値は`false`です。

#### `config.active_record.schema_cache_ignored_tables`

スキーマキャッシュの生成中に無視するテーブルのリストを定義します。テーブル名を表す文字列の`Array`または正規表現を指定できます。

#### `config.active_record.verbose_query_logs`

データベースクエリを呼び出すメソッドのソースコードの位置を、関連するクエリでログに出力するかどうかを指定します。デフォルトでは、development環境で`true`、それ以外の環境では`false`に設定されます。

#### `config.active_record.async_query_executor`

非同期クエリをプールする方法を指定します。

デフォルトは`nil`で、この場合`load_async`は無効になり、クエリをフォアグラウンドで直接実行します。
クエリを実際に非同期実行する場合は、必ず`:global_thread_pool`または`:multi_thread_pool`を指定しなければなりません。

`:global_thread_pool`: アプリケーションが接続するすべてのデータベースで単一のプールを使います。この設定は、データベースが1つしかないアプリケーションや、データベースのシャードに1度に1件しかクエリを発行しないアプリケーションに適しています。

`:multi_thread_pool`: データベースごとに1つのプールを使います。各プールのサイズは、`database.yml`ファイルの`max_threads`プロパティや`min_thread`プロパティで設定できます。この設定は、クエリを複数のデータベースに対して発行することが多いアプリケーションで、並行処理の最大数をより正確に定義したい場面で有用です。

#### `config.active_record.global_executor_concurrency`

`config.active_record.async_query_executor = :global_thread_pool`設定で、並行に実行できる非同期クエリの個数を定義します。

デフォルトは`4`です。

この数値を検討するときは、`database.yml`で設定されているデータベースプールのサイズと調和させなければなりません。コネクションプールのサイズは、フォアグラウンドのスレッド（Webサーバーやジョブワーカーのスレッド）とバックグラウンドのスレッドを両方とも扱えるサイズにする必要があります。

#### `ActiveRecord::ConnectionAdapters::Mysql2Adapter.emulate_booleans`

Active RecordのMySQLアダプタがすべての`tinyint(1)`カラムをデフォルトでbooleanと認識するかどうかを指定します。デフォルト値は`true`です。

#### `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.create_unlogged_table`

PostgreSQLが作成するデータベースを「unlogged」にすべきかどうかを制御します。unloggedにするとパフォーマンスは向上しますが、データベースがクラッシュしたときのデータ喪失リスクも増加します。production環境ではこれを有効にしないことを強くおすすめします。デフォルトではすべての環境で`false`になります。

#### `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.datetime_type`

マイグレーションやスキーマで`datetime`を呼び出したときに、Active RecordのPostgreSQLアダプタが使うネイティブ型を指定します。この設定が受け取るシンボルは、`NATIVE_DATABASE_TYPES`で設定された内容のいずれかに対応していなければなりません。デフォルト値は`:timestamp`で、この場合マイグレーション内の`t.datetime`で「タイムゾーンなしのタイムスタンプ」が作成されます。「タイムゾーン付きのタイムスタンプ」を使うには、イニシャライザで`:timestamptz`に変更します。この値を変更したときは、`bin/rails db:migrate`を実行してschema.rbをリビルドしてください。

#### `ActiveRecord::SchemaDumper.ignore_tables`

生成されるどのスキーマファイルにも**含めたくない**テーブル名の配列を渡せます。

#### `ActiveRecord::SchemaDumper.fk_ignore_pattern`

外部キー名をdb/schema.rbにダンプすべきかどうかを指定する正規表現を変更できます。デフォルトでは、`fk_rails_`で始まる外部キー名はデータベースのスキーマダンプにエクスポートされません。デフォルト値は`/^fk_rails_[0-9a-f]{10}$/`です。

### Action Controllerを設定する

`config.action_controller`には多数の設定が含まれています。

#### `config.action_controller.asset_host`

アセットを置くホストを設定します。これは、アセットをホストする場所としてアプリケーションサーバーの代りにCDN(コンテンツ配信ネットワーク)を使いたい場合に便利です。この設定を使うのは、Action Mailerで別の設定を使う場合だけにとどめてください。それ以外の場合は`config.asset_host`をお使いください。

#### `config.action_controller.perform_caching`

Action Controllerコンポーネントが提供するキャッシュ機能をアプリケーションで使うかどうかを指定します。developmentモードでは`false`、productionモードでは`true`に設定します。指定のない場合は`true`になります。

#### `config.action_controller.default_static_extension`

キャッシュされたページに与える拡張子を指定します。デフォルト値は`.html`です。

#### `config.action_controller.include_all_helpers`

すべてのビューヘルパーをあらゆる場所で使えるようにするか、対応するコントローラのスコープ内に限定するかを設定します。

`false`に設定すると、たとえば`UsersHelper`は`UsersController`の一部としてレンダリングされるビューでしか使えなくなります。`true`に設定すると、この`UsersHelper`はどこからでも使えるようになります。

デフォルト設定の振る舞い（このオプションに`true`や`false`が明示的に設定されていない場合）は、どのコントローラでもあらゆるビューヘルパーを使えます。

#### `config.action_controller.logger`

Log4rのインターフェイスまたはデフォルトのRuby Loggerクラスに従うロガーを引数として取ります。このロガーは、Action Controllerからの情報をログ出力するのに使われます。ログ出力を無効にするには`nil`を設定します。

#### `config.action_controller.request_forgery_protection_token`

RequestForgery対策用のトークンパラメータ名を設定します。`protect_from_forgery`を呼び出すと、デフォルトで`:authenticity_token`が設定されます。

#### `config.action_controller.allow_forgery_protection`

CSRF保護を有効にするかどうかを指定します。testモードではデフォルトで`false`に設定され、それ以外では`true`に設定されます。

#### `config.action_controller.forgery_protection_origin_check`

CSRFの追加対策として、HTTPの`Origin`ヘッダーがサイトのoriginと一致することをチェックすべきかどうかを設定します。

#### `config.action_controller.per_form_csrf_tokens`

CSRFトークンの正当性をそれらが生成されたメソッドやアクションに対してのみ認めるかどうかを設定します。

#### `config.action_controller.default_protect_from_forgery`

フォージェリ保護を`ActionController:Base`に追加するかどうかを指定します。

#### `config.action_controller.urlsafe_csrf_tokens`

生成されるCSRFトークンをURL-safe（URLで使ってよい文字だけを使う）にするかどうかを設定します。

#### `config.action_controller.relative_url_root`

[サブディレクトリへのデプロイ](configuring.html#サブディレクトリにデプロイする（相対url-rootの利用）)を行っていることをRailsに指示するのに使えます。デフォルト値は`ENV['RAILS_RELATIVE_URL_ROOT']`です。

#### `config.action_controller.permit_all_parameters`

マスアサインメント（mass assignment）されるすべてのパラメータをデフォルトで許可することを設定します。デフォルト値は`false`です。

#### `config.action_controller.action_on_unpermitted_parameters`

明示的に許可されていないパラメータが見つかった場合にログ出力または例外発生を行なうかどうかを指定します。test環境とdevelopment環境でのデフォルト値は`:log`であり、それ以外の環境では`false`が設定されます。以下の値を指定できます。

* `false`: 何もしない
* `:log`: `ActiveSupport::Notifications.instrument`イベントを`unpermitted_parameters.action_controller`で発火し、DEBUGレベルでログ出力する
* `:raise`: `ActionController::UnpermittedParameters`例外をraiseする

#### `config.action_controller.always_permitted_parameters`

デフォルトで許可される許可リストパラメータのリストを設定します。デフォルト値は `['controller', 'action']`です。

#### `config.action_controller.enable_fragment_cache_logging`

フラグメントキャッシュの読み書きのログを以下のように詳細な形式で出力するかどうかを指定します。

```
Read fragment views/v1/2914079/v1/2914079/recordings/70182313-20160225015037000000/d0bdf2974e1ef6d31685c3b392ad0b74 (0.6ms)
Rendered messages/_message.html.erb in 1.2 ms [cache hit]
Write fragment views/v1/2914079/v1/2914079/recordings/70182313-20160225015037000000/3b4e249ac9d168c617e32e84b99218b5 (1.1ms)
Rendered recordings/threads/_thread.html.erb in 1.5 ms [cache miss]
```

デフォルト値は`false`で、以下のように出力されます。

```
Rendered messages/_message.html.erb in 1.2 ms [cache hit]
Rendered recordings/threads/_thread.html.erb in 1.5 ms [cache miss]
```

#### `config.action_controller.raise_on_open_redirects`

許可されていないオープンリダイレクトが発生した場合に`ArgumentError`をraiseします。デフォルト値は`false`です。

#### `config.action_controller.log_query_tags_around_actions`

クエリタグのコントローラコンテキストが`around_filter`で更新されるかどうかを指定します。デフォルト値は`true`です。

#### `config.action_controller.wrap_parameters_by_default`

JSONリクエストをデフォルトで[`ParamsWrapper`](https://api.rubyonrails.org/classes/ActionController/ParamsWrapper.html)でラップするように設定します。

#### `ActionController::Base.wrap_parameters`

[`ParamsWrapper`](https://api.rubyonrails.org/classes/ActionController/ParamsWrapper.html)を設定します。これはトップレベルで呼び出すことも、コントローラで個別に呼び出すこともできます。

### Action Dispatchを設定する

#### `config.action_dispatch.session_store`

セッションデータのストア名を設定します。デフォルトのストア名は`:cookie_store`です。この他に`:active_record_store`、`:mem_cache_store`、またはカスタムクラス名なども指定できます。

#### `config.action_dispatch.default_headers`

HTTPヘッダーで使われるハッシュです。このヘッダーはデフォルトですべてのレスポンスに設定されます。このオプションは、デフォルトでは以下のように設定されます。

```ruby
config.action_dispatch.default_headers = {
  'X-Frame-Options' => 'SAMEORIGIN',
  'X-XSS-Protection' => '0',
  'X-Content-Type-Options' => 'nosniff',
  'X-Download-Options' => 'noopen',
  'X-Permitted-Cross-Domain-Policies' => 'none',
  'Referrer-Policy' => 'strict-origin-when-cross-origin'
}
```

#### `config.action_dispatch.default_charset`

すべてのレンダリングで使うデフォルトの文字セットを指定します。デフォルト値は`nil`です。

#### `config.action_dispatch.tld_length`

アプリケーションで使うトップレベルドメイン(TLD) の長さを指定します。デフォルト値は`1`です。

#### `config.action_dispatch.ignore_accept_header`

リクエストのヘッダーを受け付けるかどうかを指定します。デフォルト値は`false`です。

#### `config.action_dispatch.x_sendfile_header`

サーバー固有の`X-Sendfile`ヘッダーを指定します。これは、サーバーからの送信を加速するのに有用です。たとえば、'`X-Sendfile`をApache向けに設定できます。

#### `config.action_dispatch.http_auth_salt`

HTTP Authのsalt値（訳注: ハッシュの安全性を強化するために加えられる値）を設定します。デフォルト値は`'http authentication'`です。

#### `config.action_dispatch.signed_cookie_salt`

署名済みcookie用のsalt値を設定します。デフォルト値は`'signed cookie'`です。

#### `config.action_dispatch.encrypted_cookie_salt`

暗号化済みcookie用のsalt値を設定します。デフォルト値は`'encrypted cookie'`です。

#### `config.action_dispatch.encrypted_signed_cookie_salt`

署名暗号化済みcookie用のsalt値を設定します。デフォルト値は`'signed encrypted cookie'`です。

#### `config.action_dispatch.authenticated_encrypted_cookie_salt`

認証された暗号化済みcookieのsalt値を設定します。デフォルト値は`'authenticated encrypted cookie'`です。

#### `config.action_dispatch.encrypted_cookie_cipher`

暗号化済みcookieに使う暗号化方式を設定します。デフォルト値は`"aes-256-gcm"`です。

#### `config.action_dispatch.signed_cookie_digest`

署名済みcookieに使うダイジェスト方式を設定します。デフォルト値は`"SHA1"`です。

#### `config.action_dispatch.cookies_rotations`

署名暗号化済みcookieの秘密情報、暗号化方式、ダイジェスト方式のローテーションを行えるようにします。

#### `config.action_dispatch.use_authenticated_cookie_encryption`

署名暗号化済みcookieが値の期限切れ情報に埋め込まれる場合に、暗号化済みcookieでAES-256-GCまたは旧AES-256-CBCで認証された暗号を用いるかどうかを指定します。デフォルト値は`true`です。

#### `config.action_dispatch.use_cookies_with_metadata`

`purpose`メタデータを埋め込んだcookieの書き込みを有効にします。デフォルト値は`true`です。

#### `config.action_dispatch.perform_deep_munge`

パラメータに対して`deep_munge`メソッドを実行すべきかどうかを指定します。詳細については[セキュリティガイド](security.html#安全でないクエリ生成)を参照してください。デフォルト値は`true`です。

#### `config.action_dispatch.rescue_responses`

HTTPステータスに割り当てる例外を設定します。ここには、例外とステータスのさまざまなペアを指定したハッシュを1つ指定可能です。デフォルトの定義は次のようになっています。

```ruby
config.action_dispatch.rescue_responses = {
  'ActionController::RoutingError'
    => :not_found,
  'AbstractController::ActionNotFound'
    => :not_found,
  'ActionController::MethodNotAllowed'
    => :method_not_allowed,
  'ActionController::UnknownHttpMethod'
    => :method_not_allowed,
  'ActionController::NotImplemented'
    => :not_implemented,
  'ActionController::UnknownFormat'
    => :not_acceptable,
  'ActionController::InvalidAuthenticityToken'
    => :unprocessable_entity,
  'ActionController::InvalidCrossOriginRequest'
    => :unprocessable_entity,
  'ActionDispatch::Http::Parameters::ParseError'
    => :bad_request,
  'ActionController::BadRequest'
    => :bad_request,
  'ActionController::ParameterMissing'
    => :bad_request,
  'Rack::QueryParser::ParameterTypeError'
    => :bad_request,
  'Rack::QueryParser::InvalidParameterError'
    => :bad_request,
  'ActiveRecord::RecordNotFound'
    => :not_found,
  'ActiveRecord::StaleObjectError'
    => :conflict,
  'ActiveRecord::RecordInvalid'
    => :unprocessable_entity,
  'ActiveRecord::RecordNotSaved'
    => :unprocessable_entity
}
```

設定されていない例外はすべて500 Internel Server Errorに割り当てられます。

#### `config.action_dispatch.return_only_request_media_type_on_content_type`

`ActionDispatch::Response#content_type`が`Content-Type`ヘッダーを改変せずに返すよう変更します。

#### `config.action_dispatch.cookies_same_site_protection`

cookie設定時の`SameSite`属性のデフォルト値を設定します。`nil`に設定すると`SameSite`属性は追加されません。以下のように`proc`を渡すことでリクエストに応じて`SameSite`属性の値を動的に設定できます。

```ruby
config.action_dispatch.cookies_same_site_protection = ->(request) do
  :strict unless request.user_agent == "TestAgent"
end
```

#### `config.action_dispatch.ssl_default_redirect_status`

`ActionDispatch::SSL`ミドルウェア内で、GETリクエストとHEADリクエスト以外のリクエストをHTTPからHTTPSにリダイレクトするときに用いるデフォルトのHTTPステータスコードを設定します。デフォルト値は、[RFC7538](https://tools.ietf.org/html/rfc7538)で定義されている`308`です。

#### `config.action_dispatch.log_rescued_responses`

`rescue_responses`で設定されている、処理されなかった例外のログ出力を有効にします。デフォルト値は`true`です。

#### `ActionDispatch::Callbacks.before`

リクエストより前に実行したいコードブロックを渡します。

#### `ActionDispatch::Callbacks.after`

リクエストの後に実行したいコードブロックを渡します。

### Action Viewを設定する

`config.action_view`にも若干の設定があります。

#### `config.action_view.cache_template_loading`

リクエストのたびにビューテンプレートを再読み込みするかどうか（キャッシュしないかどうか）を指定します。デフォルト値は`config.cache_classes`に従います。

#### `config.action_view.field_error_proc`

Active Modelで発生したエラーの表示に使うHTMLジェネレータを指定します。渡したブロックは、Action Viewテンプレートのコンテキスト内で評価されます。デフォルト値は以下のとおりです。

```ruby
Proc.new { |html_tag, instance| content_tag :div, html_tag, class: "field_with_errors" }
```

#### `config.action_view.default_form_builder`

Railsでデフォルトで使うフォームビルダーを指定します。デフォルト値は、`ActionView::Helpers::FormBuilder`です。フォームビルダーを初期化処理の後に読み込みたい場合（developmentモードではフォームビルダーがリクエストのたびに再読み込みされます）は、`String`として渡すこともできます。

#### `config.action_view.logger`

Log4rのインターフェイスまたはデフォルトのRuby Loggerクラスに従うロガーを渡せます。このロガーは、Action Viewからの情報をログ出力するために使われます。ログ出力を無効にするには`nil`を設定します。

#### `config.action_view.erb_trim_mode`

ERBで使うトリムモードを指定します。デフォルト値は`'-'`で、`<%= -%>`または`<%= =%>`の場合に末尾スペースを削除して改行します。詳しくは[Erubisドキュメント](http://www.kuwata-lab.com/erubis/users-guide.06.html#topics-trimspaces)を参照してください。

#### `config.action_view.frozen_string_literal`

ERBテンプレートを`# frozen_string_literal: true`マジックコメント付きでコンパイルすることで、すべての文字列リテラルをfrozenにしてアロケーションを削減します。`true`に設定するとすべてのビューで有効になります。

#### `config.action_view.embed_authenticity_token_in_remote_forms`

フォームで`remote: true`を使う場合の`authenticity_token`のデフォルトの動作を設定します。デフォルトでは`false`で、この場合リモートフォームには`authenticity_token`フォームが含まれません。これはフォームでフラグメントキャッシュを使っている場合に便利です。

リモートフォームは`meta`タグから認証を受け取るので、JavaScriptの動作しないブラウザをサポートしなければならない場合を除いて、トークンの埋め込みは不要です。JavaScriptが動かないブラウザのサポートが必要な場合は、`authenticity_token: true`をフォームオプションとして渡すか、この設定を`true`にします。

#### `config.action_view.prefix_partial_path_with_controller_namespace`

名前空間化されたコントローラでレンダリングされたテンプレートにあるサブディレクトリから、パーシャルを探索するかどうかを指定します。たとえば、`Admin::PostsController`というコントローラがあり、以下のテンプレートを出力するとします。

```erb
<%= render @article %>
```

デフォルト設定は`true`で、その場合`/admin/posts/_post.erb`にあるパーシャルを使います。この値を`false`にすると、`/posts/_post.erb`がレンダリングされます。この動作は、`PostsController`などの名前空間化されていないコントローラでレンダリングした場合と同じです。

#### `config.action_view.automatically_disable_submit_tag`

クリック時に`submit_tag`を自動的に無効にするかどうかを指定します。デフォルト値は`true`です。

#### `config.action_view.debug_missing_translation`

訳文の存在しないキーを`<span>`タグで囲むかどうかを指定します。デフォルト値は`true`です。

#### `config.action_view.form_with_generates_remote_forms`

`form_with`でリモートフォームを生成するかどうかを指定します。

#### `config.action_view.form_with_generates_ids`

`form_with`でidを生成するかどうかを指定します。

#### `config.action_view.default_enforce_utf8`

フォームを生成するときに、UTF-8でエンコードされたフォームを古いInternet Explorerで強制送信する隠しタグを付けるかどうかを指定します。デフォルト値は`false`です。

#### `config.action_view.image_loading`

`image_tag`ヘルパーでレンダリングされた`<img>`タグの`loading`属性のデフォルト値を指定します。

たとえば`"lazy"`を設定すると、`image_tag`ヘルパーでレンダリングされた`<img>`タグに`loading="lazy"`が含まれ、[画像がビューポートに近づくまで読み込みを遅延するようブラウザに指示します](https://html.spec.whatwg.org/#lazy-loading-attributes)（`image_tag`に`loading: "eager"`渡すなどの方法で、画像ごとに挙動を上書きできます）。デフォルト値は`nil`です。

#### `config.action_view.image_decoding`

`image_tag`ヘルパーでレンダリングされる`<img>`タグの`decoding`属性に使うデフォルト値を指定します。デフォルト値は`nil`です。

#### `config.action_view.annotate_rendered_view_with_filenames`

レンダリングされたビューにテンプレートファイル名を追加するかどうかを指定します。デフォルト値は`false`です。

#### `config.action_view.preload_links_header`

`javascript_include_tag`や`stylesheet_link_tag`で、アセットをプリロードする`Link`ヘッダーを生成するかどうかを指定します。

#### `config.action_view.button_to_generates_button_tag`

`button_to`で、コンテンツが第1引数またはブロックとして渡されるかどうかにかかわらず、`<button>`要素をレンダリングするかどうかを指定します。

#### `config.action_view.apply_stylesheet_media_default`

`media`属性が提供されていない場合に、`stylesheet_link_tag`で`media`属性のデフォルト値を`screen`としてレンダリングするかどうかを指定します。

### Action Mailboxを設定する

`config.action_mailbox`には以下の設定オプションがあります。

#### `config.action_mailbox.logger`

Action Mailboxで用いるロガーを含みます。Log4rまたはデフォルトのRuby Loggerクラスに従うロガーを渡せます。デフォルト値は`Rails.logger`です。

```ruby
config.action_mailbox.logger = ActiveSupport::Logger.new(STDOUT)
```

#### `config.action_mailbox.incinerate_after`

`ActiveSupport::Duration`を受け取ります。これは`ActionMailbox::InboundEmail`レコードを処理後に自動的に破棄（destroy）するまでの期間を指定します。デフォルト値は`30.days`です。

```ruby
# 受信メールの処理後、14日後に「焼却」する
config.action_mailbox.incinerate_after = 14.days
```

#### `config.action_mailbox.queues.incineration`

焼却（incinerate）ジョブに用いるActive Jobキューを示すシンボルを渡せます。このオプションが`nil`の場合、焼却ジョブがデフォルトのActive Jobキューに送信されます（`config.active_job.default_queue_name`を参照）。

#### `config.action_mailbox.queues.routing`

ルーティングジョブに用いるActive Jobキューを示すシンボルを渡せます。このオプションが`nil`の場合、ルーティングジョブがデフォルトのActive Jobキューに送信されます（`config.active_job.default_queue_name`を参照）。

#### `config.action_mailbox.storage_service`

メールのアップロードに使うActive Storageサービスを示すシンボルで指定します。このオプションが`nil`の場合、メールがデフォルトのActive Storageサービスにアップロードされます（`config.active_storage.service`を参照）。

### Action Mailerを設定する

`config.action_mailer`には多数の設定オプションがあります。

#### `config.action_mailer.asset_host`

メーラーで用いるアセットのホストを指定します。アプリケーションサーバーではなくCDNにアセットをホスティングする場合に便利です。この設定はAction Controllerで異なるアセットホストを設定する場合にのみ使います。それ以外の場合は`config.asset_host`をお使いください。

#### `config.action_mailer.logger`

Log4rのインターフェイスまたはデフォルトのRuby Loggerクラスに従うロガーを引数として取ります。このロガーは、Action Mailerからの情報をログ出力するために使われます。ログ出力を無効にするには`nil`を設定します。

#### `config.action_mailer.smtp_settings`

`:smtp`配信方法を詳細に設定するのに使えます。引数に渡すオプションハッシュには、以下のオプションを含められます。

* `:address`: リモートのメールサーバーを指定します。デフォルトの"localhost"設定から変更します。
* `:port`: 使うメールサーバーのポートが25番でない場合は（めったにないと思いますが）、ここで対応できます。
* `:domain`: HELOドメインの指定が必要な場合に使います。
* `:user_name`: メールサーバーで認証が要求される場合は、ここでユーザー名を設定します。
* `:password`: メールサーバーで認証が要求される場合は、ここでパスワードを設定します。
* `:authentication`: メールサーバーで認証が要求される場合は、ここで認証の種類を指定します。`:plain`、`:login`、`:cram_md5`のいずれかのシンボルを指定できます。
* `:enable_starttls`: SMTPサーバーにSTARTTLSで接続します（サポートされていない場合は失敗します）。デフォルト値は`false`です。
* `:enable_starttls_auto`: 利用するSMTPサーバーでSTARTTLSが有効かどうかを検出し、可能な場合は使います。デフォルト値は`true`です。
* `:openssl_verify_mode`: TLSを使う場合、OpenSSLの認証方法を設定できます。これは、自己署名証明書やワイルドカード証明書が必要な場合に便利です。OpenSSLの検証定数である`:none`や`:peer`を指定することも、`OpenSSL::SSL::VERIFY_NONE`定数や`OpenSSL::SSL::VERIFY_PEER`定数を直接指定することもできます。
* `:ssl/:tls`: SMTP接続でSMTP/TLS（SMTPS: SMTP over direct TLS connection）を有効にします。
* `:open_timeout`: コネクション開始の試行中の待ち時間を秒で指定します。
* `:read_timeout`: read(2)呼び出しのタイムアウトを秒で指定します。

また、[`Mail::SMTP`をサポートする任意の設定オプション](https://github.com/mikel/mail/blob/master/lib/mail/network/delivery_methods/smtp.rb)も渡せます。

#### `config.action_mailer.smtp_timeout`

メール配信用の`:smtp`メソッドの`:open_timeout`値と`:read_timeout`値を設定できます。

#### `config.action_mailer.sendmail_settings`

`:sendmail`の詳細な配信方法を設定できます。引数に渡すオプションハッシュには、以下のオプションを含められます。

* `:location` - sendmail実行ファイルの場所。デフォルト値は`/usr/sbin/sendmail`です。
* `:arguments` - コマンドラインに与える引数。デフォルト値は`-i`です。

#### `config.action_mailer.raise_delivery_errors`

メール配信が完了しなかった場合にエラーを発生させるかどうかを指定します。デフォルト値は`true`です。

#### `config.action_mailer.delivery_method`

メール配信方法を指定します。デフォルト値は`:smtp`です。詳しくは[Action Mailerガイド](action_mailer_basics.html#action-mailerを設定する)を参照してください。

#### `config.action_mailer.perform_deliveries`

メールを実際に配信するかどうかを指定します。デフォルト値は`true`です。テスト時にメール送信を抑制するのに便利です。

#### `config.action_mailer.default_options`

Action Mailerのデフォルトを設定します。これは、メーラーごとに`from`や`reply_to`などを設定します。デフォルト値は以下のとおりです。

```ruby
mime_version:  "1.0",
charset:       "UTF-8",
content_type: "text/plain",
parts_order:  ["text/plain", "text/enriched", "text/html"]
```

ハッシュを1つ指定してオプションを追加することもできます。

```ruby
config.action_mailer.default_options = {
  from: "noreply@example.com"
}
```

#### `config.action_mailer.observers`

メールを配信したときに通知を受けるオブザーバーを登録します。

```ruby
config.action_mailer.observers = ["MailObserver"]
```

#### `config.action_mailer.interceptors`

メールを送信する前に呼び出すインターセプタを登録します。

```ruby
config.action_mailer.interceptors = ["MailInterceptor"]
```

#### `config.action_mailer.preview_interceptors`

メールのプレビュー前に呼び出すインターセプタを登録します。

```ruby
config.action_mailer.preview_interceptors = ["MyPreviewMailInterceptor"]
```

#### `config.action_mailer.preview_path`

メーラーのプレビュー場所を指定します

```ruby
config.action_mailer.preview_path = "#{Rails.root}/lib/mailer_previews"
```

#### `config.action_mailer.show_previews`

メーラーのプレビューを有効または無効にします。デフォルトではdevelopment環境で`true`です。

```ruby
config.action_mailer.show_previews = false
```

#### `config.action_mailer.deliver_later_queue_name`

配信ジョブで用いるActive Jobキューを指定します。

このオプションが`nil`に設定されている場合、配信ジョブはデフォルトのActive Jobキューに送信されます（`config.active_job.default_queue_name`を参照）。Active Jobアダプタが指定のキューを処理可能な設定になっていることも確認してください。そうしない場合、配信ジョブはエラーを出さずに無視される可能性があります。

#### `config.action_mailer.perform_caching`

メーラーのテンプレートでフラグメントキャッシュを有効にするかどうかを指定します。指定のない場合のデフォルト値は`true`です。

#### `config.action_mailer.delivery_job`

メールの配信ジョブを指定します。

### Active Supportを設定する

Active Supportにもいくつかの設定オプションがあります。

#### `config.active_support.bare`

Rails起動時に`active_support/all`の読み込みを行なうかどうかを指定します。デフォルト値は`nil`であり、この場合`active_support/all`が読み込まれます。

#### `config.active_support.test_order`

テストケースの実行順序を指定します。`:random`か`:sorted`を指定可能で、デフォルト値は`:random`です。

#### `config.active_support.escape_html_entities_in_json`

JSONシリアライズに含まれるHTMLエンティティをエスケープするかどうかを指定します。デフォルト値は`true`です。

#### `config.active_support.use_standard_json_time_format`

日付のシリアライズをISO 8601フォーマットで行うかどうかを指定します。デフォルト値は`true`です。

#### `config.active_support.time_precision`

JSONエンコードされた時間値の精度を指定します。デフォルト値は`3`桁です。

#### `config.active_support.hash_digest_class`

重要でないダイジェスト（ETagヘッダーなど）の生成に用いるダイジェスト用クラスを設定します。

#### `config.active_support.key_generator_hash_digest_class`

暗号化済みcookieなどで、設定済みのsecretを元にsecretを導出するのに用いるダイジェスト用クラスを設定します。

#### `config.active_support.use_authenticated_message_encryption`

AES-256-CBCではなくAES-256-GCM認証済み暗号を用いるかどうかを指定します。

#### `config.active_support.cache_format_version`

利用するキャッシュシリアライザのバージョンを指定します。指定可能な値は`6.1`と`7.0`です。

#### `config.active_support.deprecation`

非推奨警告メッセージの振る舞いを設定します。`:raise`、`:stderr`、`:log`、`:notify`、`:silence`を指定可能です。デフォルト値は`:stderr`です。`ActiveSupport::Deprecation.behavior`でも設定可能です。

#### `config.active_support.disallowed_deprecation`

利用が許されない非推奨警告メッセージの振る舞いを設定します。`:raise`、`:stderr`、`:log`、`:notify`、`:silence`を指定可能です。デフォルト値は`:raise`です。`ActiveSupport::Deprecation.disallowed_behavior`でも設定可能です。

#### `config.active_support.disallowed_deprecation_warnings`

アプリケーションで利用を許可しない項目として扱う非推奨警告メッセージを設定します。これを用いて、たとえば特定の非推奨項目を重大な失敗として扱えるようになります。`ActiveSupport::Deprecation.disallowed_warnings`でも設定可能です。

#### `config.active_support.report_deprecations`

利用が許されない非推奨項目も含めて、すべての非推奨警告メッセージを停止できます。`ActiveSupport::Deprecation.warn`の設定は無効になります。production環境ではデフォルトで有効になります。

#### `config.active_support.isolation_level`

Rails内部ステートのほとんどの局所性（locality）を設定します。Fiberベースのサーバーやジョブプロセッサ（`falcon`など）を使う場合は、`:fiber`を設定してください。
それ以外の場合は局所性を`:thread`にするのが最適です。

#### `config.active_support.use_rfc4122_namespaced_uuids`

生成される名前空間化UUIDで、`Digest::UUID.uuid_v3`メソッドや`Digest::UUID.uuid_v5`メソッドに`String`として渡す名前空間IDをRFC 4122標準に準拠させるかどうかを指定します。

`true`に設定する場合:

* 名前空間IDにはUUIDのみが許される。許されていない名前空間IDが渡されると`ArgumentError`が発生する。
* 使われる名前空間が`Digest::UUID`で定義された定数または`String`の場合に、非推奨警告メッセージを生成しない。
* 名前空間IDは大文字小文字を区別しない。
* 生成される名前空間化UUIDはすべて標準に準拠する。

`false`に設定する場合:

* 任意の`String`値を名前空間IDに利用可能（ただし推奨されない）。互換性維持のため、`ArgumentError`エラーは発生しない。
* 渡される名前空間IDが、`Digest::UUID`で定義される定数でない場合は非推奨警告メッセージを生成する。
* 名前空間IDは大文字小文字が区別される。
* `Digest::UUID`で定義される名前空間ID定数を用いて生成される名前空間化UUIDのみが標準に準拠する。

新規アプリのデフォルト値は`true`です。アップグレードされるアプリでは、後方互換性用に`false`が設定されます。

#### `config.active_support.executor_around_test_case`

テストケースをラップする`Rails.application.executor.wrap`を呼び出すように設定します。これにより、テストケースの振る舞いが実際のリクエストやジョブに近づきます。Active Recordクエリキャッシュや非同期クエリのような、通常のテストで無効にされる多くの機能が有効になります。

#### `config.active_support.disable_to_s_conversion`

ある種のRubyコアクラスに含まれる`#to_s`メソッドの上書きを無効にします。この設定は、アプリケーションで[Ruby 3.1の最適化](https://github.com/ruby/ruby/commit/b08dacfea39ad8da3f1fd7fdd0e4538cc892ec44)をいち早く利用したい場合に使えます。

この設定は、`config/application.rb`の`Application`クラス内に記述する必要があります。それ以外の場所では無効です。

#### `ActiveSupport::Logger.silencer`

`false`に設定すると、ブロック内でのログ出力を抑制する機能がオフになります。デフォルト値は`true`です。

#### `ActiveSupport::Cache::Store.logger`

キャッシュストア操作で使うロガーを指定します。

#### `ActiveSupport.utc_to_local_returns_utc_offset_times`

`ActiveSupport::TimeZone.utc_to_local`で、オフセットを考慮したUTC時間ではなく、UTCオフセットを考慮したローカル時間を返すように設定します。

### Active Jobを設定する

`config.active_job`では以下の設定オプションが利用できます。

#### `config.active_job.queue_adapter`

キューのバックエンドに用いるアダプタを設定します。デフォルトのアダプタは`:async`です。最新の組み込みアダプタについてはAPIドキュメント[`ActiveJob::QueueAdapters`](https://api.rubyonrails.org/classes/ActiveJob/QueueAdapters.html)を参照してください。

```ruby
# 必ずGemfileにアダプタのgemを追加し、
# アダプタ固有のインストール/デプロイ方法に従うこと
config.active_job.queue_adapter = :sidekiq
```

#### `config.active_job.default_queue_name`

デフォルトのキュー名を変更できます。デフォルト値は`"default"`です。

```ruby
config.active_job.default_queue_name = :medium_priority
```

#### `config.active_job.queue_name_prefix`

すべてのジョブ名の前に付けられるプレフィックスを設定します（スペースは含めません）。デフォルト値は空欄なので何も追加されません。

以下の設定では、production実行時に指定のジョブが`production_high_priority`キューに送信されます。

```ruby
config.active_job.queue_name_prefix = Rails.env
```

```ruby
class GuestsCleanupJob < ActiveJob::Base
  queue_as :high_priority
  #....
end
```

#### `config.active_job.queue_name_delimiter`

デフォルト値は`'_'`です。`queue_name_prefix`が設定されている場合は、プレフィックスされていないキュー名とプレフィックスの結合に`queue_name_delimiter`が使われます。

以下の設定では、指定のジョブが`video_server.low_priority`キューに送信されます。

```ruby
# この区切り文字を使うにはprefixを設定しなければならない
config.active_job.queue_name_prefix = 'video_server'
config.active_job.queue_name_delimiter = '.'
```

```ruby
class EncoderJob < ActiveJob::Base
  queue_as :low_priority
  #....
end
```

#### `config.active_job.logger`

Active Jobのログ情報に使うロガーとして、Log4rのインターフェイスに準拠したロガーか、デフォルトのRubyロガーを指定できます。このロガーは、Active JobのクラスかActive Jobのインスタンスで`logger`を呼び出すことで取り出せます。ログ出力を無効にするには`nil`を設定します。

#### `config.active_job.custom_serializers`

カスタムの引数シリアライザを設定できます。デフォルト値は`[]`です。

#### `config.active_job.log_arguments`

ジョブの引数をログに出力するかどうかを指定します。デフォルト値は`true`です。

#### `config.active_job.retry_jitter`

失敗したジョブをリトライするときに算出する遅延時間に加えるジッター（jitter: ランダムな微変動値）の総量を指定します。

#### `config.active_job.log_query_tags_around_perform`

クエリタグのジョブコンテキストが`around_perform`で自動的に更新されるようにするかどうかを指定します。デフォルト値は`true`です。

### Action Cableを設定する

#### `config.action_cable.url`

Action CableサーバーがホストされているURLを文字列で指定します。Action Cableサーバーがメインのアプリケーションと別になっている場合に使う可能性があります。

#### `config.action_cable.mount_path`

Action Cableをメインサーバープロセスの一部としてマウントする場所を文字列で指定します。デフォルト値は`/cable`です。`nil`を設定すると、Action Cableは通常のRailsサーバーの一部としてマウントされなくなります。

設定オプションについて詳しくは、[Action Cableの概要](action_cable_overview.html#設定)を参照してください。

### Active Storageを設定する

`config.active_storage`では以下の設定オプションが提供されています。

#### `config.active_storage.variant_processor`

`:mini_magick`または`:vips`いずれかのシンボルを渡せます。これらはvariantの変換やblob解析にMiniMagickとruby-vipsのどちらを使うかを指定します。デフォルト値は`:mini_magick`です。

#### `config.active_storage.analyzers`

Active Storageのblob（binary large object）で利用できるアナライザを指定するクラスの配列を受け取ります。デフォルトでは以下のように定義されます。

```ruby
config.active_storage.analyzers = [ActiveStorage::Analyzer::ImageAnalyzer::Vips, ActiveStorage::Analyzer::ImageAnalyzer::ImageMagick, ActiveStorage::Analyzer::VideoAnalyzer, ActiveStorage::Analyzer::AudioAnalyzer]
```

画像アナライザは、画像blobの幅（width）や高さ（height）を取り出せます。

動画アナライザは、動画blobの幅（width）、高さ（height）、再生時間（duration）、角度（angle）、アスペクト比（aspect ratio）、動画/音声チャンネルの有無を取り出せます。

音声アナライザは、音声blobの再生時間やビットレートを取り出せます。

#### `config.active_storage.previewers`

Active Storageのblobで利用できる画像プレビューアを指定するクラスを配列で受け取ります。デフォルトでは以下のように定義されます。

```ruby
config.active_storage.previewers = [ActiveStorage::Previewer::PopplerPDFPreviewer, ActiveStorage::Previewer::MuPDFPreviewer, ActiveStorage::Previewer::VideoPreviewer]
```

`PopplerPDFPreviewer`と`MuPDFPreviewer`はPDF blobの最初のページのサムネイルを生成できます。

`VideoPreviewer`は動画blobのフレームの中から動画の内容を代表するフレームを生成できます。

#### `config.active_storage.paths`

プレビューアやアナライザのコマンドがあるディレクトリを示すオプションをハッシュで受け取ります。デフォルトの`{}`の場合、コマンドをデフォルトパスで探索します。オプションには以下を含められます。

* `:ffprobe`: ffprobe実行ファイルの場所
* `:mutool`: mutool実行ファイルの場所
* `:ffmpeg`: ffmpeg実行ファイルの場所

```ruby
config.active_storage.paths[:ffprobe] = '/usr/local/bin/ffprobe'
```

#### `config.active_storage.variable_content_types`

Active StorageがImageMagickに変換可能なcontent typeを示す文字列を配列で受け取ります。デフォルトでは以下のように定義されます。

```ruby
config.active_storage.variable_content_types = %w(image/png image/gif image/jpeg image/tiff image/vnd.adobe.photoshop image/vnd.microsoft.icon image/webp image/avif image/heic image/heif)
```

#### `config.active_storage.web_image_content_types`

variantをフォールバック用のPNGフォーマットに変換せずに処理可能なWeb画像Content-Typeを示す文字列を配列で受け取ります。アプリケーションのvariant処理に`WebP`や`AVIF`を使いたい場合は、この配列に`image/webp`や`image/avif`を追加できます。デフォルトでは以下のように定義されます。

```ruby
config.active_storage.web_image_content_types = %w(image/png image/jpeg image/gif)
```

#### `config.active_storage.content_types_to_serve_as_binary`

Active Storageが常に添付ファイルとして扱うContent-Typeを示す文字列を配列で受け取ります。デフォルトでは以下のように定義されます。

```ruby
config.active_storage.content_types_to_serve_as_binary = %w(text/html image/svg+xml application/postscript application/x-shockwave-flash text/xml application/xml application/xhtml+xml application/mathml+xml text/cache-manifest)
```

#### `config.active_storage.content_types_allowed_inline`

Active Storageでインライン配信を許可するContent-Typeを示す文字列を配列で受け取ります。デフォルトでは以下のように定義されます。

```ruby
config.active_storage.content_types_allowed_inline` = %w(image/png image/gif image/jpeg image/tiff image/vnd.adobe.photoshop image/vnd.microsoft.icon application/pdf)
```

#### `config.active_storage.silence_invalid_content_types_warning`

Rails 7以降は、Rails 6で誤ってサポートされていた無効なContent-Typeを使うとActive Storageで警告メッセージが表示されます。この警告メッセージは以下の設定でオフにできます。

```ruby
config.active_storage.silence_invalid_content_types_warning = false
```

#### `config.active_storage.queues.analysis`

解析ジョブに用いるActive Jobキューをシンボルで指定します。このオプションが`nil`の場合、解析ジョブはデフォルトのActive Jobキューに送信されます（`config.active_job.default_queue_name`を参照）。

```ruby
config.active_storage.queues.analysis = :low_priority
```

#### `config.active_storage.queues.purge`

purgeジョブに用いるActive Jobキューをシンボルで指定します。このオプションが`nil`の場合、purgeジョブはデフォルトのActive Jobキューに送信されます（`config.active_job.default_queue_name`を参照）。

```ruby
config.active_storage.queues.purge = :low_priority
```

#### `config.active_storage.queues.mirror`

ダイレクトアップロードのミラーリングジョブに用いるActive Jobキューをシンボルで指定します。このオプションが`nil`の場合、ミラーリングジョブはデフォルトのActive Jobキューに送信されます（`config.active_job.default_queue_name`を参照）。デフォルト値は`nil`です。

```ruby
config.active_storage.queues.mirror = :low_priority
```

#### `config.active_storage.logger`

Active Storageで用いられるロガーを設定できます。Log4rのインターフェイスに沿ったロガーや、デフォルトのRuby `Logger`クラスを指定できます。

```ruby
config.active_storage.logger = ActiveSupport::Logger.new(STDOUT)
```

#### `config.active_storage.service_urls_expire_in`

以下によって生成されるURLのデフォルトの有効期限を指定します。

* `ActiveStorage::Blob#url`
* `ActiveStorage::Blob#service_url_for_direct_upload`
* `ActiveStorage::Variant#url`

デフォルト値は5分間です。

#### `config.active_storage.urls_expire_in`

Active Storageで生成される、Railsアプリケーション内URLのデフォルトの有効期限を指定します。デフォルト値は`nil`です。

#### `config.active_storage.routes_prefix`

Active Storageが提供するルーティングのプレフィックスを設定できます。生成されるルーティングの冒頭に追加する文字列を渡せます。

```ruby
config.active_storage.routes_prefix = '/files'
```

デフォルト値は`/rails/active_storage`です。

#### `config.active_storage.replace_on_assign_to_many`

`has_many_attached`で宣言された添付ファイルのコレクションに代入するときに、既存の添付ファイルをすべて置き換えるか、追加（append）するかを指定します。デフォルト値は`true`です。

#### `config.active_storage.track_variants`

variantをデータベースに記録するかどうかを指定します。デフォルト値は`true`です。

#### `config.active_storage.draw_routes`

Active Storageのルーティング生成をオンオフできます。デフォルト値は`true`です。

#### `config.active_storage.resolve_model_to_route`

Active Storageのファイル配信方法をグローバルに変更できます。

利用可能な値は以下です。

* `:rails_storage_redirect`: 署名済みの短命なサービスURLにリダイレクトする
* `:rails_storage_proxy`: プロキシファイルでダウンロードする

デフォルト値は`:rails_storage_redirect`です。

#### `config.active_storage.video_preview_arguments`

ffmpegの動画プレビュー画像生成方法を変更できます。

`config.load_defaults 7.0`の場合はデフォルトで以下が定義されます。

```ruby
config.active_storage.video_preview_arguments = "-vf 'select=eq(n\\,0)+eq(key\\,1)+gt(scene\\,0.015),loop=loop=-1:size=2,trim=start_frame=1' -frames:v 1 -f image2"
```

振る舞いは以下のようになります。

1. `select=eq(n\,0)+eq(key\,1)+gt(scene\,0.015)`: 冒頭の動画フレームとキーフレーム、閾値に合う場面変更のフレーム
2. `loop=loop=-1:size=2,trim=start_frame=1`: 他のフレームが条件を満たさない場合は冒頭の動画フレームにフォールバックする（選択したフレームの冒頭の1つまたは2つをループさせてから、冒頭のループフレームを削除する）

#### `config.active_storage.multiple_file_field_include_hidden`

Rails 7.1以降、Active Storageの`has_many_attached`リレーションシップは、デフォルトで現在のコレクションに**追加されるのではなく**、デフォルトで現在のコレクションを**置き換える**ようになる予定です。「**空の**」コレクションの送信をサポートするには、Action Viewのフォームビルダーでチェックボックス要素をレンダリングするのと同じ要領で、補助的な隠しフィールドをレンダリングしてください。

### Action Textを設定する

#### `config.action_text.attachment_tag_name`

添付ファイルをラップするHTMLタグを文字列で指定します。デフォルト値は`"action-text-attachment"`です。

### `load_defaults`の結果

`config.load_defaults`は、渡されたバージョンまでのデフォルト値を含む新しいデフォルト値を設定します。たとえば`6.0`を指定すると、6.0以前のあらゆるバージョンのデフォルト値も取得できます。

#### '7.0'を指定した場合（以前のバージョンのデフォルト値を除く）

- `config.action_controller.raise_on_open_redirects`: `true`
- `config.action_view.button_to_generates_button_tag`: `true`
- `config.action_view.apply_stylesheet_media_default`: `false`
- `config.active_support.key_generator_hash_digest_class`: `OpenSSL::Digest::SHA256`
- `config.active_support.hash_digest_class`: `OpenSSL::Digest::SHA256`
- `config.active_support.cache_format_version`: `7.0`
- `config.active_support.remove_deprecated_time_with_zone_name`: `true`
- `config.active_support.executor_around_test_case`: `true`
- `config.active_support.use_rfc4122_namespaced_uuids`: `true`
- `config.active_support.disable_to_s_conversion`: `true`
- `config.action_dispatch.return_only_request_media_type_on_content_type`: `false`
- `config.action_dispatch.cookies_serializer`: `:json`
- `config.action_mailer.smtp_timeout`: `5`
- `config.active_storage.video_preview_arguments`: `"-vf 'select=eq(n\\,0)+eq(key\\,1)+gt(scene\\,0.015),loop=loop=-1:size=2,trim=start_frame=1' -frames:v 1 -f image2"`
- `config.active_storage.multiple_file_field_include_hidden`: `true`
- `config.active_record.automatic_scope_inversing`: `true`
- `config.active_record.verify_foreign_keys_for_fixtures`: `true`
- `config.active_record.partial_inserts`: `false`
- `config.active_storage.variant_processor`: `:vips`
- `config.action_controller.wrap_parameters_by_default`: `true`
- `config.action_dispatch.default_headers`:

```ruby
    {
      "X-Frame-Options" => "SAMEORIGIN",
      "X-XSS-Protection" => "0",
      "X-Content-Type-Options" => "nosniff",
      "X-Download-Options" => "noopen",
      "X-Permitted-Cross-Domain-Policies" => "none",
      "Referrer-Policy" => "strict-origin-when-cross-origin"
    }
```

#### '6.1'を指定した場合（以前のバージョンのデフォルト値を除く）

- `config.active_record.has_many_inversing`: `true`
- `config.active_record.legacy_connection_handling`: `false`
- `config.active_storage.track_variants`: `true`
- `config.active_storage.queues.analysis`: `nil`
- `config.active_storage.queues.purge`: `nil`
- `config.action_mailbox.queues.incineration`: `nil`
- `config.action_mailbox.queues.routing`: `nil`
- `config.action_mailer.deliver_later_queue_name`: `nil`
- `config.active_job.retry_jitter`: `0.15`
- `config.action_dispatch.cookies_same_site_protection`: `:lax`
- `config.action_dispatch.ssl_default_redirect_status` = `308`
- `ActiveSupport.utc_to_local_returns_utc_offset_times`: `true`
- `config.action_controller.urlsafe_csrf_tokens`: `true`
- `config.action_view.form_with_generates_remote_forms`: `false`
- `config.action_view.preload_links_header`: `true`

#### '6.0'を指定した場合（以前のバージョンのデフォルト値を除く）

- `config.autoloader`: `:zeitwerk`
- `config.action_view.default_enforce_utf8`: `false`
- `config.action_dispatch.use_cookies_with_metadata`: `true`
- `config.action_mailer.delivery_job`: `"ActionMailer::MailDeliveryJob"`
- `config.active_storage.queues.analysis`: `:active_storage_analysis`
- `config.active_storage.queues.purge`: `:active_storage_purge`
- `config.active_storage.replace_on_assign_to_many`: `true`
- `config.active_record.collection_cache_versioning`: `true`

#### '5.2'を指定した場合（以前のバージョンのデフォルト値を除く）

- `config.active_record.cache_versioning`: `true`
- `config.action_dispatch.use_authenticated_cookie_encryption`: `true`
- `config.active_support.use_authenticated_message_encryption`: `true`
- `config.active_support.hash_digest_class`: `OpenSSL::Digest::SHA1`
- `config.action_controller.default_protect_from_forgery`: `true`
- `config.action_view.form_with_generates_ids`: `true`

#### '5.1'を指定した場合（以前のバージョンのデフォルト値を除く）

- `config.assets.unknown_asset_fallback`: `false`
- `config.action_view.form_with_generates_remote_forms`: `true`

#### '5.0'を指定した場合（ベースラインのデフォルト値を除く）

- `config.action_controller.per_form_csrf_tokens`: `true`
- `config.action_controller.forgery_protection_origin_check`: `true`
- `ActiveSupport.to_time_preserves_timezone`: `true`
- `config.active_record.belongs_to_required_by_default`: `true`
- `config.ssl_options`: `{ hsts: { subdomains: true } }`

#### 基本のデフォルト値

- `config.action_controller.default_protect_from_forgery`: `false`
- `config.action_controller.raise_on_open_redirects`: `false`
- `config.action_controller.urlsafe_csrf_tokens`: `false`
- `config.action_dispatch.cookies_same_site_protection`: `nil`
- `config.action_mailer.delivery_job`: `ActionMailer::MailDeliveryJob`
- `config.action_view.form_with_generates_ids`: `false`
- `config.action_view.preload_links_header`: `nil`
- `config.action_view.button_to_generates_button_tag`: `false`
- `config.action_view.apply_stylesheet_media_default`: `true`
- `config.active_job.retry_jitter`: `0.0`
- `config.action_mailbox.queues.incineration`: `:action_mailbox_incineration`
- `config.action_mailbox.queues.routing`: `:action_mailbox_routing`
- `config.action_mailer.deliver_later_queue_name`: `:mailers`
- `config.active_record.collection_cache_versioning`: `false`
- `config.active_record.cache_versioning`: `false`
- `config.active_record.has_many_inversing`: `false`
- `config.active_record.legacy_connection_handling`: `true`
- `config.active_record.partial_inserts`: `true`
- `config.active_support.use_authenticated_message_encryption`: `false`
- `config.active_support.hash_digest_class`: `OpenSSL::Digest::MD5`
- `config.active_support.key_generator_hash_digest_class`: `OpenSSL::Digest::SHA1`
- `config.active_support.cache_format_version`: `6.1`
- `config.active_support.executor_around_test_case`: `false`
- `config.active_support.isolation_level`: `:thread`
- `config.active_support.use_rfc4122_namespaced_uuids`: `false`
- `config.active_support.disable_to_s_conversion`: `false`
- `config.action_dispatch.return_only_request_media_type_on_content_type`: `true`
- `ActiveSupport.utc_to_local_returns_utc_offset_times`: `false`
- `config.action_mailer.smtp_timeout`: `nil`
- `config.active_storage.video_preview_arguments`: `"-y -vframes 1 -f image2"`
- `config.active_storage.multiple_file_field_include_hidden`: `false`
- `config.active_storage.variant_processor`: `:mini_magick`
- `config.action_controller.wrap_parameters_by_default`: `false`
- `config.action_dispatch.default_headers`:

```ruby
    {
      "X-Frame-Options" => "SAMEORIGIN",
      "X-XSS-Protection" => "1; mode=block",
      "X-Content-Type-Options" => "nosniff",
      "X-Download-Options" => "noopen",
      "X-Permitted-Cross-Domain-Policies" => "none",
      "Referrer-Policy" => "strict-origin-when-cross-origin"
    }
```

### データベースを設定する

ほぼすべてのRailsアプリケーションは、何らかの形でデータベースにアクセスします。データベースへの接続は、環境変数`ENV['DATABASE_URL']`を設定するか、`config/database.yml`というファイルを設定することで行えます。

`config/database.yml`ファイルを使うことで、データベース接続に必要なすべての情報を指定できます。

```yaml
development:
  adapter: postgresql
  database: blog_development
  pool: 5
```

上の設定は、`postgresql`を用いて`blog_development`という名前のデータベースに接続します。同じ接続情報をURL化して、以下のように環境変数に保存することも可能です。

```ruby
> puts ENV['DATABASE_URL']
postgresql://localhost/blog_development?pool=5
```

`config/database.yml`ファイルには、Railsがデフォルトで実行できる以下の3つの異なる環境を記述するセクションが含まれています。

* `development`環境: ローカルの開発環境でアプリケーションと手動でやりとりを行うために使われます。
* `test`環境: 自動化されたテストを実行するために使われます。
* `production`環境: アプリケーションを世界中に公開する本番環境で使われます。

必要であれば、`config/database.yml`の内部でURLを直接指定することも可能です。

```ruby
development:
  url: postgresql://localhost/blog_development?pool=5
```

`config/database.yml`ファイルにはERBタグ`<%= %>`も含められます。タグ内に記載されたものはすべてRubyのコードとして評価されます。このタグを用いて、環境変数から接続情報を取り出したり、接続情報の生成に必要な計算を行なうことも可能です。

TIP: データベースの接続設定を手動で更新する必要はありません。アプリケーションのジェネレータのオプションを表示してみると、`--database`というオプションがあるのがわかります。このオプションでは、リレーショナルデータベースで最もよく使われるアダプタをリストから選択できます。さらに、`cd .. && rails new blog --database=mysql`のようにジェネレータを繰り返し実行することも可能です。`config/database.yml`ファイルが上書きされたことを確認すれば、アプリケーションの設定はSQLite用からMySQL用に変更されます。よく使われるデータベース接続方法の詳しい例については後述します。

### 接続設定

データベース接続の設定方法は、`config/database.yml`による方法と環境変数による方法の2とおりがあります。この2つがどのように相互作用するかを理解しておくことが重要です。

`config/database.yml`ファイルの内容が空で、かつ環境変数`ENV['DATABASE_URL']`が設定されている場合、データベースへの接続には環境変数が使われます。

```bash
$ cat config/database.yml

$ echo $DATABASE_URL
postgresql://localhost/my_database
```

`config/database.yml`ファイルがあり、環境変数`ENV['DATABASE_URL']`が設定されていない場合は、`config/database.yml`ファイルがデータベース接続に使われます。

```bash
$ cat config/database.yml
development:
  adapter: postgresql
  database: my_database
  host: localhost

$ echo $DATABASE_URL
```

`config/database.yml`ファイルと環境変数`ENV['DATABASE_URL']`が両方存在する場合、両者の設定はマージして使われます。以下のいくつかの例を参照して理解を深めてください。

提供された接続情報が重複している場合、環境変数が優先されます。

```bash
$ cat config/database.yml
development:
  adapter: sqlite3
  database: NOT_my_database
  host: localhost

$ echo $DATABASE_URL
postgresql://localhost/my_database

$ bin/rails runner 'puts ActiveRecord::Base.configurations'
#<ActiveRecord::DatabaseConfigurations:0x00007fd50e209a28>

$ bin/rails runner 'puts ActiveRecord::Base.configurations.inspect'
#<ActiveRecord::DatabaseConfigurations:0x00007fc8eab02880 @configurations=[
  #<ActiveRecord::DatabaseConfigurations::UrlConfig:0x00007fc8eab020b0
    @env_name="development", @spec_name="primary",
    @config={"adapter"=>"postgresql", "database"=>"my_database", "host"=>"localhost"}
    @url="postgresql://localhost/my_database">
  ]
```

上の実行結果で使われているアダプタ、ホスト、データベースは`ENV['DATABASE_URL']`の内容と一致しています。

提供された複数の情報が重複ではなく競合している場合も、常に環境変数の接続設定が優先されます。

```bash
$ cat config/database.yml
development:
  adapter: sqlite3
  pool: 5

$ echo $DATABASE_URL
postgresql://localhost/my_database

$ bin/rails runner 'puts ActiveRecord::Base.configurations'
#<ActiveRecord::DatabaseConfigurations:0x00007fd50e209a28>

$ bin/rails runner 'puts ActiveRecord::Base.configurations.inspect'
#<ActiveRecord::DatabaseConfigurations:0x00007fc8eab02880 @configurations=[
  #<ActiveRecord::DatabaseConfigurations::UrlConfig:0x00007fc8eab020b0
    @env_name="development", @spec_name="primary",
    @config={"adapter"=>"postgresql", "database"=>"my_database", "host"=>"localhost", "pool"=>5}
    @url="postgresql://localhost/my_database">
  ]
```

poolは`ENV['DATABASE_URL']`で提供される情報に含まれていないので、マージされています。adapterは重複しているので、`ENV['DATABASE_URL']`の接続情報が優先されています。

`ENV['DATABASE_URL']`の情報よりもdatabase.ymlの情報を優先する唯一の方法は、database.ymlで`"url"`サブキーを用いて明示的にURL接続を指定することです。

```bash
$ cat config/database.yml
development:
  url: sqlite3:NOT_my_database

$ echo $DATABASE_URL
postgresql://localhost/my_database

$ bin/rails runner 'puts ActiveRecord::Base.configurations'
#<ActiveRecord::DatabaseConfigurations:0x00007fd50e209a28>

$ bin/rails runner 'puts ActiveRecord::Base.configurations.inspect'
#<ActiveRecord::DatabaseConfigurations:0x00007fc8eab02880 @configurations=[
  #<ActiveRecord::DatabaseConfigurations::UrlConfig:0x00007fc8eab020b0
    @env_name="development", @spec_name="primary",
    @config={"adapter"=>"sqlite3", "database"=>"NOT_my_database"}
    @url="sqlite3:NOT_my_database">
  ]
```

今度は`ENV['DATABASE_URL']`の接続情報は無視されました。アダプタとデータベース名が異なります。

`config/database.yml`にはERBを記述できるので、database.yml内で明示的に`ENV['DATABASE_URL']`を使うのが最もよい方法です。これは特にproduction環境で有用です。理由は、データベース接続のパスワードのような秘密情報をGitなどのソースコントロールに直接登録すべきではないからです。

```bash
$ cat config/database.yml
production:
  url: <%= ENV['DATABASE_URL'] %>
```

以上の説明で動作が明らかになりました。接続情報は決してdatabase.ymlに直接書かず、常に`ENV['DATABASE_URL']`に保存したものを利用してください。

#### SQLite3データベースを設定する

Railsには[SQLite3](http://www.sqlite.org)のサポートが組み込まれています。SQLiteは軽量かつ専用サーバーの不要なデータベースアプリケーションです。SQLiteは開発用・テスト用であれば問題なく使えますが、（訳注: 同時アクセスが多い）本番での利用には耐えられない可能性があります。Railsで新規プロジェクトを作成するとデフォルトでSQLiteが指定されますが、これはいつでも後から変更できます。

以下はデフォルトの接続設定ファイル(`config/database.yml`)に含まれる、開発環境用の接続設定です。

```yaml
development:
  adapter: sqlite3
  database: db/development.sqlite3
  pool: 5
  timeout: 5000
```

NOTE: Railsでデータ保存用にSQLite3データベースが採用されている理由は、設定なしですぐに使えるからです。RailsではSQLiteの他にMySQL（MariaDB含む）やPostgreSQLなども使えますし、データベース接続用のプラグインも多数あります。production環境で何らかのデータベースを使う場合、そのためのアダプタはたいていの場合探せば見つかります。

#### MySQLやMariaDBデータベースを設定する

Rails同梱のSQLite3ではなくMySQLやMariaDBなどを採用する場合、`config/database.yml`の記述方法を少し変更します。developmentセクションの記述は以下のようになります。

```yaml
development:
  adapter: mysql2
  encoding: utf8mb4
  database: blog_development
  pool: 5
  username: root
  password:
  socket: /tmp/mysql.sock
```

ユーザー名root、パスワードなしでdevelopment環境のデータベースに接続できれば、上の設定で接続できるはずです。接続できない場合は、`development`セクションのユーザー名またはパスワードを適切なものに変更してください。

NOTE: MySQLのバージョンが5.5または5.6で、かつ`utf8mb4`文字セットをデフォルトで使いたい場合は、MySQLサーバーで`innodb_large_prefix`システム変数を有効にすることで、長いキープレフィックスがサポートされるよう設定してください。

MySQLのAdvisory Locksはデフォルトで有効になります。これはデータベースマイグレーションの並行処理を安全に実行するために用いられます。`advisory_locks`を`false`にするとAdvisory Locksを無効にできます。

```yaml
production:
  adapter: mysql2
  advisory_locks: false
```

#### PostgreSQLデータベースを設定する

PostgreSQLを採用した場合は、`config/database.yml`の記述は以下のようになります。

```yaml
development:
  adapter: postgresql
  encoding: unicode
  database: blog_development
  pool: 5
```

Active Recordでは、Prepared StatementsやAdvisory Locksなどの機能がデフォルトでオンになります。PgBouncerなどの外部コネクションプーラーを用いる場合、これらの機能をオフにできます。

```yaml
production:
  adapter: postgresql
  prepared_statements: false
  advisory_locks: false
```

オンにする場合、Active Recordはデフォルトでデータベース接続ごとに最大`1000`までのPrepared Statementsを作成します。この数値を変更したい場合は`statement_limit`に別の数値を指定します。

```yaml
production:
  adapter: postgresql
  statement_limit: 200
```

Prepared Statementsの利用量を増やすと、データベースで必要なメモリー量も増大します。PostgreSQLデータベースのメモリー利用量が上限に達した場合は、`statement_limit`の値を小さくするかPrepared Statementsをオフにしてください。

#### JRubyプラットフォームでSQLite3データベースを設定する

JRuby環境でSQLite3を採用する場合、`config/database.yml`の記述方法は少し異なります。developmentセクションは以下のようになります。

```yaml
development:
  adapter: jdbcsqlite3
  database: db/development.sqlite3
```

#### JRubyプラットフォームでMySQLやMariaDBのデータベースを使う

JRuby環境でMySQLやMariaDBなどを採用する場合、`config/database.yml`の記述方法は少し異なります。developmentセクションは以下のようになります。

```yaml
development:
  adapter: jdbcmysql
  database: blog_development
  username: root
  password:
```

#### JRubyプラットフォームでPostgreSQLデータベースを使う

JRuby環境でPostgreSQLを採用する場合、`config/database.yml`の記述方法は少し異なります。developmentセクションは以下のようになります。

```yaml
development:
  adapter: jdbcpostgresql
  encoding: unicode
  database: blog_development
  username: blog
  password:
```

`development`セクションのユーザー名とパスワードは適切なものに置き換えてください。

#### メタデータストレージを設定する

Railsは、自分のRails環境とスキーマに関する情報を、`ar_internal_metadata`という名前の内部テーブルにデフォルトで保存します。

この機能をコネクションごとにオフにするには、利用するデータベース設定ファイルで以下のように`use_metadata_table`を設定します。これは、共有データベースで作業する場合や、テーブル作成権限を持たないデータベースユーザーで作業する場合に便利です。

```yaml
development:
  adapter: postgresql
  use_metadata_table: false
```

### Rails環境を作成する

Railsにデフォルトで備わっている環境は、"development"、"test"、"production"の3つです。通常はこの3つの環境で事足りますが、場合によっては環境を追加したくなることもあると思います。

たとえば、production環境をミラーコピーしたサーバーをテスト目的でのみ使いたいという場合を想定してみましょう。このようなサーバーは通常「ステージングサーバー（staging server）」と呼ばれます。"staging"環境をサーバーに追加したいのであれば、`config/environments/staging.rb`というファイルを作成するだけで済みます。その際にはなるべく`config/environments`にある既存のファイルを流用し、必要な部分のみを変更するようにしてください。

このようにして追加された環境は、デフォルトの3つの環境と同じように利用できます。`rails server -e staging`を実行すればステージング環境でサーバーを起動でき、`rails console -e staging`や`Rails.env.staging?`なども動作するようになります。

### サブディレクトリにデプロイする（相対URL rootの利用）

Railsアプリケーションの実行は、アプリケーションのrootディレクトリ (`/`など) で行なうことが前提となっています。この節では、アプリケーションをディレクトリの下で実行する方法について説明します。

ここでは、アプリケーションを"/app1"ディレクトリにデプロイしたいとします。これを行なうには、適切なルーティングを生成できるディレクトリをRailsに指示する必要があります。

```ruby
config.relative_url_root = "/app1"
```

あるいは、`RAILS_RELATIVE_URL_ROOT`環境変数に設定することも可能です。

これで、リンクが生成される時に"/app1"がディレクトリ名の前に追加されます。

#### Passengerを使う

Passengerを使うと、アプリケーションを手軽にサブディレクトリで実行できます。設定方法について詳しくは、[passengerマニュアル](https://www.phusionpassenger.com/library/deploy/apache/deploy/ruby/#deploying-an-app-to-a-sub-uri-or-subdirectory)を参照してください。

#### リバースプロキシを使う

リバースプロキシを用いるアプリケーションをデプロイすることで、従来のデプロイと比べて確実なメリットが得られます。アプリケーションで必要なコンポーネントの層が追加され、サーバーを制御しやすくなります。

現代的なWebサーバーの多くは、キャッシュサーバーやアプリケーションサーバーなどのロードバランシングにプロキシサーバーを用いています。

[Unicorn](https://bogomips.org/unicorn/)は、リバースプロキシの背後で実行されるアプリケーションサーバーの例です。

この場合、NGINXやApacheなどのプロキシサーバーを設定して、アプリケーションサーバー（ここではUnicorn）からの接続を受け付けるようにする必要があります。Unicornは、デフォルトでTCP接続のポート8000をリッスンしますが、このポート番号を変更することも、ソケットを用いるように設定することも可能です。

詳しくは[Unicorn readme](https://bogomips.org/unicorn/README.html)を参照し、背後の[哲学](https://bogomips.org/unicorn/PHILOSOPHY.html)を理解してください。

アプリケーションサーバーの設定が終わったら、Webサーバーも適切に設定してリクエストのプロキシを行わなければなりません。以下の設定はNGINXの設定に含まれることがあります。


```nginx
upstream application_server {
  server 0.0.0.0:8080;
}

server {
  listen 80;
  server_name localhost;

  root /root/path/to/your_app/public;

  try_files $uri/index.html $uri.html @app;

  location @app {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://application_server;
  }

  # （省略）
}
```

必ず[NGINXのドキュメント](https://nginx.org/en/docs/)で最新情報を参照してください。

Rails環境の設定
--------------------------

一部の設定については、Railsの外部から環境変数の形で与えることも可能です。以下の環境変数は、Railsの多くの部分で認識されます。

* `ENV["RAILS_ENV"]`: Railsが実行される環境 (production、development、testなど) を定義します。

* `ENV["RAILS_RELATIVE_URL_ROOT"]`: [アプリケーションをサブディレクトリにデプロイする](configuring.html#サブディレクトリにデプロイする（相対url-rootの利用）)ときにルーティングシステムがURLを認識するために使われます。

* `ENV["RAILS_CACHE_ID"]`と`ENV["RAILS_APP_VERSION"]`: Railsのキャッシュを扱うコードで拡張キャッシュを生成するために使われます。これにより、１つのアプリケーションの中で複数の独立したキャッシュを扱えるようになります。

イニシャライザファイルを使う
-----------------------

Railsは、フレームワークの読み込みとすべてのgemの読み込みが完了してから、イニシャライザの読み込みを開始します。イニシャライザとは、アプリケーションの`config/initializers`ディレクトリに保存されているRubyファイルのことです。

イニシャライザファイルには、フレームワークやgemがすべて読み込まれた後に行いたい設定（フレームワークやgemを設定するオプションなど）を保存できます。

NOTE: 自分のイニシャライザが、他のすべてのgemのイニシャライザが実行された後で実行されるという保証はありません。そのようなgemに依存する初期化コードは、`config.after_initialize`ブロックに配置してください。

初期化イベント
---------------------

Railsにはフック可能な初期化イベントが5つあります。以下のイベントは、実際に実行される順序で掲載しています。

#### `before_configuration`

このフックは、アプリケーションが`Rails::Application`から定数を継承した直後に実行されます。`config`呼び出しは、このイベントより前に評価されます。

#### `before_initialize`

このフックは、`:bootstrap_hook`イニシャライザによる初期化プロセスの直前に直接実行されます。`:bootstrap_hook`は、Railsアプリケーション初期化プロセスのうち比較的初期の段階で実行されます。

#### `to_prepare`

このフックは、Railtiesの初期化処理とアプリケーション自身の初期化処理がすべて完了した後で、かつeager loadingとミドルウェアスタックの構築が行われる前に実行されます。さらに重要な点は、このフックは`development`モードではサーバーへのリクエストのたびに必ず実行されますが、`production`モードと`test`モードでは起動時に1度しか実行されないことです。

#### `before_eager_load`

このフックは、事前eager loadingの前に直接実行されます。これは`production`環境のデフォルトの動作で、`development`環境では実行されません。

#### `after_initialize`

このフックは、アプリケーションの初期化が完了し、かつ`config/initializers`以下のイニシャライザが実行された後に実行されます。

これらのフックでイベントを定義するには、`Rails::Application`、`Rails::Railtie`、または`Rails::Engine`サブクラス内でブロック記法を使います。

```ruby
module YourApp
  class Application < Rails::Application
    config.before_initialize do
      # ここに初期化コードを書く
    end
  end
end
```

あるいは、`Rails.application`オブジェクトで以下のように`config`メソッドを実行することも可能です。

```ruby
Rails.application.config.before_initialize do
  # ここに初期化コードを書く
end
```

WARNING: アプリケーションの一部（特にルーティング）には、`after_initialize`ブロックが呼び出された時点で設定が完了しないものがあります。

### `Rails::Railtie#initializer`

Railsには起動時に実行されるさまざまなイニシャライザがあり、それらはすべて`Rails::Railtie`の`initializer`メソッドで定義されます。以下はAction Controllerの`set_helpers_path`イニシャライザの例です。

```ruby
initializer "action_controller.set_helpers_path" do |app|
  ActionController::Helpers.helpers_path = app.helpers_paths
end
```

この`initializer`メソッドは3つの引数を取ります。第1引数はイニシャライザ名、第2引数はオプションハッシュ（上の例では使っていません）、そして第3引数はブロックです。

オプションハッシュに含まれる`:before`キーを使うと、新しいイニシャライザより前に実行したいイニシャライザを指定できます。同様に、`:after`キーを使うと、このイニシャライザより**後**に実行したいイニシャライザを指定できます。

`initializer`メソッドで定義されたイニシャライザは、定義された順序で実行されます（`:before`や`:after`を使った場合は除きます）。

WARNING: イニシャライザの起動順序は、論理的に矛盾が生じない限り、どのイニシャライザの前または後にでも配置可能です。たとえば、"one"〜"four"という4つのイニシャライザがあり、かつこの順序で定義されたとします。ここで"four"を"two"より**前**かつ"three"よりも**後**になるように定義すると論理矛盾が発生し、イニシャライザの実行順を決定できなくなってしまいます。

`initializer`メソッドのブロック引数は、アプリケーション自身のインスタンスです。そのおかげで、上の例で示したように、`config`メソッドでアプリケーションの設定にアクセスできます。

`Rails::Application`は`Rails::Railtie`を間接的に継承しているので、`config/application.rb`で`initializer`メソッドを使ってアプリケーションの初期化処理を定義できます。

### イニシャライザ

以下はRailsにあるイニシャライザのリストです。これらは定義された順序で並んでおり、特記事項がない限り実行されます。

#### `load_environment_hook`

これはプレースホルダとして使われます。具体的には、`:load_environment_config`を定義してこのイニシャライザより前に実行したい場合に使います。

#### `load_active_support`

Active Supportの基本部分を設定する`active_support/dependencies`を`require`します。`config.active_support.bare`がない場合はデフォルトで`active_support/all`を`require`します。

#### `initialize_logger`

ここより前の位置に`Rails.logger`を定義するイニシャライザがない場合、アプリケーションのロガー（`ActiveSupport::Logger`オブジェクト）を初期化し、`Rails.logger`にアクセスできるようにします。

#### `initialize_cache`

`Rails.cache`が未設定の場合、`config.cache_store`の値を参照してキャッシュを初期化し、その結果を`Rails.cache`として保存します。そのオブジェクトが`middleware`メソッドに応答する場合、そのミドルウェアをミドルウェアスタックの`Rack::Runtime`の前に挿入します。

#### `set_clear_dependencies_hook`

このイニシャライザは、`cache_classes`が`false`の場合にのみ実行され、オブジェクト空間からのリクエスト中に参照された定数を`ActionDispatch::Callbacks.after`で削除します。これにより、これらの定数が以後のリクエストで再読み込みされるようになります。

#### `bootstrap_hook`

このフックはすべての設定済み`before_initialize`ブロックを実行します。

#### `i18n.callbacks`

development環境の場合、`to_prepare`コールバックを設定します。このコールバックは、最後にリクエストが発生した後にロケールが変更されると`I18n.reload!`を呼び出します。productionモードの場合、このコールバックは最初のリクエストでのみ実行されます。

#### `active_support.deprecation_behavior`

環境に応じた非推奨項目レポートをセットアップします。デフォルト値は、development環境では`:log`、production環境では`:silence`、test環境では`:stderr`です。値は配列で設定できます。

このイニシャライザは、許可しない非推奨項目の扱いについても設定します。デフォルト値は、development環境では`:raise`、production環境では`:silence`です。許可しない非推奨警告のデフォルト値は、空の配列です。

#### `active_support.initialize_time_zone`

`config.time_zone`の設定に基いてアプリケーションのデフォルトタイムゾーンを設定します。デフォルト値は"UTC"です。

#### `active_support.initialize_beginning_of_week`

`config.beginning_of_week`の設定に基づいてアプリケーションのデフォルトの週開始日を設定します。デフォルト値は`:monday`です。

#### `active_support.set_configs`

`config.active_support`内の設定を用いてActive Supportをセットアップします。メソッド名を`ActiveSupport`のセッターとして`send`し、その値をActive Supportに渡します。

#### `action_dispatch.configure`

`ActionDispatch::Http::URL.tld_length`を構成して、`config.action_dispatch.tld_length`の値（トップレベルドメイン名の長さ）が設定されるようにします。

#### `action_view.set_configs`

`config.action_view`内の設定を用いてAction Viewをセットアップします。メソッド名を`ActionView::Base`のセッターとして`send`し、その値をAction Viewに渡します。

#### `action_controller.assets_config`

明示的に設定されていない場合は、`config.action_controller.assets_dir`をアプリケーションの`public/`ディレクトリに設定します。

#### `action_controller.set_helpers_path`

Action Controllerの`helpers_path`をアプリケーションの`helpers_path`に設定します。

#### `action_controller.parameters_config`

`ActionController::Parameters`で使うStrong Parametersオプションを設定します。

#### `action_controller.set_configs`

`config.action_controller`内の設定を用いてAction Controllerをセットアップします。メソッド名を`ActionController::Base`のセッターとして`send`し、その値をAction Controllerに渡します。

#### `action_controller.compile_config_methods`

指定された設定用メソッドを初期化し、より高速にアクセスできるようにします。

#### `active_record.initialize_timezone`

`ActiveRecord::Base.time_zone_aware_attributes`を`true`に設定し、`ActiveRecord::Base.default_timezone`をUTCに設定します。属性がデータベースから読み込まれると、`Time.zone`で指定されたタイムゾーンに変換されます。

#### `active_record.logger`

`ActiveRecord::Base.logger`に`Rails.logger`を設定します（未設定の場合）。

#### `active_record.migration_error`

未実行のマイグレーションがあるかどうかをチェックするミドルウェアを設定します。

#### `active_record.check_schema_cache_dump`

スキーマキャッシュダンプを読み込みます（設定済みかつ可能な場合）。

#### `active_record.warn_on_records_fetched_greater_than`

クエリから返されたレコード数が非常に多い場合の警告を有効にします。

#### `active_record.set_configs`

`config.active_record`内の設定を用いてActive Recordをセットアップします。メソッド名を`ActiveRecord::Base`のセッターとして`send`し、その値をActive Recordに渡します。

#### `active_record.initialize_database`

データベース設定を`config/database.yml`（デフォルト）から読み込み、現在の環境でデータベース接続を確立します。

#### `active_record.log_runtime`

リクエストでActive Record呼び出しに要した時間をロガーに出力する`ActiveRecord::Railties::ControllerRuntime`をインクルードします。

#### `active_record.set_reloader_hooks`

`config.cache_classes`が`false`の場合、再読み込み可能なデータベース接続をすべてリセットします。

#### `active_record.add_watchable_files`

変更の監視対象ファイルに`schema.rb`ファイルと`structure.sql`ファイルを追加します。

#### `active_job.logger`

`ActiveJob::Base.logger`に`Rails.logger`を設定します（未設定の場合）。

#### `active_job.set_configs`

`config.active_job`内の設定を用いてActive Jobをセットアップします。メソッド名を`ActiveJob::Base`のセッターとして`send`し、その値をActive Jobに渡します。

#### `action_mailer.logger`

`ActionMailer::Base.logger`に`Rails.logger`を設定します（未設定の場合）。

#### `action_mailer.set_configs`

`config.action_mailer`内の設定を用いてAction Mailerをセットアップします。メソッド名を`ActionMailer::Base`のセッターとして`send`し、その値をAction Mailerに渡します。

#### `action_mailer.compile_config_methods`

指定された設定用メソッドを初期化し、より高速にアクセスできるようにします。

#### `set_load_path`

このイニシャライザは`bootstrap_hook`の前に実行されます。`config.load_paths`およびすべての自動読み込みパスが`$LOAD_PATH`に追加されます。

#### `set_autoload_paths`

このイニシャライザは`bootstrap_hook`の前に実行されます。`app/`以下のすべてのサブディレクトリと、`config.autoload_paths`、`config.eager_load_paths`、`config.autoload_once_paths`で指定したすべてのパスが`ActiveSupport::Dependencies.autoload_paths`に追加されます。

#### `add_routing_paths`

デフォルトで、アプリケーションやrailtiesやエンジンにある`config/routes.rb`ファイルをすべて読み込み、アプリケーションのルーティングを設定します。

#### `add_locales`

アプリケーションやrailtiesやエンジンにある`config/locales`ファイルを`I18n.load_path`に追加し、そのパスで指定された場所にある訳文にアクセスできるようにします。

#### `add_view_paths`

`app/views`（アプリケーションとrailties、エンジンも含む）へのパスをビューファイルへの探索パスに追加します。

#### `load_environment_config`

現在の環境に対応する`config/environments`を読み込みます。

#### `prepend_helpers_path`

アプリケーションやrailtiesやエンジンに含まれる`app/helpers`ディレクトリをヘルパーへの探索パスに追加します。

#### `load_config_initializers`

アプリケーションやrailtiesやエンジンにある`config/initializers`のRubyファイルをすべて読み込みます。このディレクトリに置かれているファイルには、フレームワークの読み込みがすべて完了した後に行うべき設定も保存できます。

#### `engines_blank_point`

エンジンの読み込みが完了する前に行いたい処理に使う初期化ポイントへのフックを提供します。このポイント以後は、railtiesやエンジンのイニシャライザはすべて実行されます。

#### `add_generator_templates`

アプリケーションやrailtiesやエンジンにある`lib/templates`ディレクトリにあるジェネレータ用のテンプレートを探索して`config.generators.templates`設定に追加します。この設定によって、すべてのジェネレータからテンプレートを参照できるようになります。

#### `ensure_autoload_once_paths_as_subset`

`config.autoload_once_paths`に、`config.autoload_paths`以外のパスが含まれないようにします。それ以外のパスが含まれている場合は例外が発生します。

#### `add_to_prepare_blocks`

アプリケーションやrailtiesやエンジンにあるすべての`config.to_prepare`呼び出しのブロックが、Action Dispatchの`to_prepare`に追加されます。Action Dispatchはdevelopmentモードではリクエストごとに実行され、productionモードでは最初のリクエストの前に実行されます。

#### `add_builtin_route`

アプリケーションがdevelopment環境で動作している場合、`rails/info/properties`へのルーティングをアプリケーションのルーティングに追加します。ブラウザでこのルーティングにアクセスすると、デフォルトのRailsアプリケーションで`public/index.html`に表示されるのと同様の詳細情報（RailsやRubyのバージョンなど）を表示できます。

#### `build_middleware_stack`

アプリケーションのミドルウェアスタックをビルドし、`call`メソッドを持つオブジェクトを返します。この`call`メソッドは、リクエストに対するRack環境のオブジェクトを受け取ります。

#### `eager_load!`

`config.eager_load`がtrueに設定されている場合、`config.before_eager_load`フックを実行してから`eager_load!`を呼び出します。この呼び出しによって、すべての`config.eager_load_namespaces`が呼び出されます。

#### `finisher_hook`

アプリケーションの初期化プロセス完了後に実行されるフックを提供します。アプリケーションやrailtiesやエンジンの`config.after_initialize`ブロックもすべて実行します。

#### `set_routes_reloader_hook`

ルーティングファイルが`ActiveSupport::Callbacks.to_run`で再読み込みされるようAction Dispatchを構成します。

#### `disable_dependency_loading`

`config.eager_load`が`true`の場合は自動依存関係読み込み（automatic dependency loading）を無効にします。

データベース接続をプールする
----------------

Active Recordのデータベース接続は`ActiveRecord::ConnectionAdapters::ConnectionPool`で管理されます。これは、コネクション数に限りのあるデータベース接続にアクセスするときに、スレッドアクセス数とコネクションプールが同期するようにします。デフォルトの最大接続数は5で、`database.yml`でカスタマイズ可能です。

```ruby
development:
  adapter: sqlite3
  database: db/development.sqlite3
  pool: 5
  timeout: 5000
```

コネクションプールはデフォルトでActive Recordが扱うので、ThinやPumaやUnicornなどのアプリケーションサーバーの動作は同じになるはずです。最初はデータベースのコネクションプールは空で、コネクションプールの上限に達するまで必要に応じてコネクションが追加されます。

１つのリクエストでは、データベースアクセスが最初に必要になったときにコネクションをチェックアウト（貸出）し、リクエストが終了するときにコネクションをチェックイン（返却）します。つまり、キューで待機している後続のリクエストで追加のコネクションスロットが再び利用可能になります。

利用可能な数を超えるコネクションを使おうとすると、Active Recordはコネクションをブロックし、プールのコネクションが空くのを待ちます。コネクションを取得できない場合は以下のようなタイムアウトエラーが発生します。

```ruby
ActiveRecord::ConnectionTimeoutError - could not obtain a database connection within 5.000 seconds (waited 5.000 seconds)
```

上のエラーが発生する場合は、`database.yml`の`pool`オプションの数値を増やしてコネクションプールのサイズを増やすとよいでしょう。

NOTE: マルチスレッド環境で動作しているアプリケーションでは、多数のスレッドが多数のコネクションに同時アクセスする可能性があります。その時点のリクエストの負荷によっては、限られたコネクションを多数のスレッドが奪い合う可能性があります。

カスタム設定
--------------------

Railsの設定オブジェクトに独自のコードを設定するには、`config.x`名前空間または`config`に直接コードを書きます。両者の重要な違いは、**ネストした**設定（`config.x.nested.nested.hi`など）の場合は`config.x`を使うべきで、**単一レベル**の設定（`config.hello`など）では`config`だけを使うべきであるという点です。

```ruby
config.x.payment_processing.schedule = :daily
config.x.payment_processing.retries  = 3
config.super_debugger = true
```

これにより、設定オブジェクトを介してこれらの設定場所にアクセス可能になります。

```ruby
Rails.configuration.x.payment_processing.schedule # => :daily
Rails.configuration.x.payment_processing.retries  # => 3
Rails.configuration.x.payment_processing.not_set  # => nil
Rails.configuration.super_debugger                # => true
```

`Rails::Application.config_for`を使うと、設定ファイル全体を読み込むことも可能です。

```yaml
# config/payment.yml
production:
  environment: production
  merchant_id: production_merchant_id
  public_key:  production_public_key
  private_key: production_private_key

development:
  environment: sandbox
  merchant_id: development_merchant_id
  public_key:  development_public_key
  private_key: development_private_key
```

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.payment = config_for(:payment)
  end
end
```

```ruby
Rails.configuration.payment['merchant_id'] # => production_merchant_id or development_merchant_id
```

`Rails::Application.config_for`は、共通の設定をグループ化する`shared`設定をサポートしています。この共有設定は、環境設定にマージされます。

```yaml
# config/example.yml
shared:
  foo:
    bar:
      baz: 1

development:
  foo:
    bar:
      qux: 2
```

```ruby
# development environment
Rails.application.config_for(:example)[:foo][:bar] #=> { baz: 1, qux: 2 }
```

検索エンジン向けのインデックス作成
-----------------------

場合によっては、アプリケーションの一部のページをGoogleやBingやYahooやDuck Duck Goなどの検索サイトに表示したくないことがあります。サイトのインデックスを作成するロボットは、インデックス作成を許可されているページを調べるために、最初に`http://your-site.com/robots.txt`ファイルの内容を分析します。

Railsはこのファイルを`/public`の下に作成します。デフォルトの設定では、アプリケーションのすべてのページで検索エンジンによるインデックス作成を許可します。アプリケーションのすべてのページでインデックス作成をブロックするには、robots.txtに以下を記述します。

```
User-agent: *
Disallow: /
```

特定のページのみをブロックする場合は、もう少し複雑な構文が必要です。詳しくはrobot.txtの[公式ドキュメント](https://www.robotstxt.org/robotstxt.html)を参照してください。

イベントベースのファイルシステム監視
---------------------------

[listen](https://github.com/guard/listen) gemを使うと、イベントベースのファイルシステム監視を利用してRailsのファイル変更を検出できます（`config.cache_classes`が`false`の場合）。

```ruby
group :development do
  gem 'listen', '~> 3.3'
end
```

listen gemを使わない場合、Railsはリクエストのたびにファイルの変更の有無を検出するためにアプリケーションのツリーをすべて探索します。

LinuxやmacOSでは追加のgemは不要ですが、[*BSD](https://github.com/guard/listen#on-bsd)や[Windows](https://github.com/guard/listen#on-windows)環境では追加のソフトウェアが必要になることがあります。

[一部の設定がサポート対象外](https://github.com/guard/listen#issues--limitations)である点にご注意ください。
