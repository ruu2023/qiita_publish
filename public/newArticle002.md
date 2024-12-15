---
title: LaravelでCRUDしたいので情報を集める
tags:
  - Laravel
  - Vue.js
  - vite
private: false
updated_at: '2024-12-15T10:38:03+09:00'
id: 1385c70187faa6fe5349
organization_url_name: null
slide: false
ignorePublish: false
---

# この記事で書くこと

- Laravel + Vite(vue)で scss
- Laravel の CRUD
- model の作成
- バリデーションメッセージの日本語化

## scss の導入

```bash
src# npm add -D sass-embedded
```

## model と migration の同時作成

```bash
src# php artisan make:model Book --migration
```

- migration の書き方

https://qiita.com/Takahiro_Nago/items/71d30873313862ab6818

```php
    public function up(): void
    {
        Schema::create('skill_lists', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id'); //外部キー
            $table->unsignedBigInteger('skill_id'); //外部キー
            $table->integer('level');
            $table->integer('exp');
            $table->timestamps();

            $table->foreign('user_id')->references('id')->on('users');
            $table->foreign('skill_id')->references('id')->on('skills');
        });
    }
```

- 指定したファイルの migration

```bash
src# php artisan migrate --path=/database/migrations/2024_12_08_053317_create_skills_table.php
```

- seeder の作成

```bash
src# php artisan make:seeder SkillsTableSeeder
```

- seeder の編集

```/database/seeders/SkillListsTableSeeder.php
use Illuminate\Support\Facades\DB;

...
    public function run(): void
    {
        DB::table('skill_lists')->insert([
            [
                'user_id' => 1,
                'skill_id' => 1,
                'level' => 2,
                'exp' => 200,
                'created_at' => "2024-12-09"
            ],
            [
                'user_id' => 1,
                'skill_id' => 2,
                'level' => 3,
                'exp' => 300,
                'created_at' => "2024-12-09"
            ],
            [
                'user_id' => 2,
                'skill_id' => 1,
                'level' => 4,
                'exp' => 100,
                'created_at' => "2024-12-09"
            ],
        ]);
    }
```

```/database/seeders/DatabaseSeeder.php
public function run(): void {
  $this->call(SkillListsTableSeeder::class);
}
```

- 指定した seeder の実行

```bash
src# php artisan db:seed --class=CalendarsTableSeeder
```

- timezone の設定（今更）

```env:.env
APP_TIMEZONE=Asia/Tokyo

// app.phpが読んで適用する
```

## crud 処理 (UPDATE と DELETE はまた今度)

- 今回の ER 図
  ![er.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/1c40c73a-8696-d9e8-a65f-97ef97a3254c.png)

- fillable で追加を許可する、hasMany は 1 対多の関係（function 名は複数）

```php:app/Models/skills.php
use Illuminate\Database\Eloquent\Relations\HasMany;

class Skill extends Model
{

    protected $fillable = [
        'name',
        'created_at'
    ];

    public function skill_lists(): HasMany
    {
        return $this->hasMany(SkillList::class);
    }
    public function calendars(): HasMany
    {
        return $this->hasMany(Calendar::class);
    }
}
```

```php:app/Models/Calendar.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Calendar extends Model
{

    protected $fillable = [
        'user_id',
        'skill_id',
        'exp',
        'created_at',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
    public function skill(): BelongsTo
    {
        return $this->belongsTo(Skill::class);
    }
}
```

- belongsTo の書き方（function 名は単数）

```php:app/Models/SkillList.php
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class SkillList extends Model
{
    protected $fillable = [
        'user_id',
        'skill_id',
        'exp',
        'created_at',
    ];
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
    public function skill(): BelongsTo
    {
        return $this->belongsTo(Skill::class);
    }
}
```

- controller の作成

```bash
/src# php artisan make:controller CalendarController
```

```php:app/Http/Controllers/CalendarController.php
class CalendarController extends Controller
{
    public function index(Request $request)
    {
        // 主←従 (READ)
        // $skillLists = SkillList::find(3)->user->name;
        // $skillLists = "a";
        // dd($skillLists);
    }

    public function create()
    {
        return view('calendar.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'name' => 'required|max:25'
        ]);
        Skill::create([
            'name' => $request->input('name'),
            'created_at' => now(),
        ]);
        return redirect()->route('calendar.index')
            ->with('success', 'Post created successfully.');
    }
}
```

- routing の編集

```php:routes/web.php
use App\Http\Controllers\CalendarController;

Route::get('/calendar/create', CalendarController::class . '@create')->name('calendar.create');
// adds a post to the database
Route::post('/calendar', CalendarController::class . '@store')->name('calendar.store');
// returns a page that shows a full post
```

- view の作成

```bash
src# php artisan make:view calendar.create
```

```php:resources/views/calendar.blade.php
<div class="container h-100 mt-5">
  <div class="row h-100 justify-content-center align-items-center">
    <div class="col-10 col-md-8 col-lg-6">
      <h3>Add a skill</h3>
      <form action="{{ route('calendar.store') }}" method="post">
        @csrf
        <div class="form-group">
          <label for="name">name</label>
          <input type="text" class="form-control" id="name" name="name" required>
        </div>
        <br>
        <button type="submit" class="btn btn-primary">Create skill</button>
      </form>
      //エラーハンドリング
      @if ($errors->any())
        <div class="text-red-500">
            <ul>
                @foreach ($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
            </ul>
        </div>
      @endif
    </div>
  </div>
</div>
```

- localhost/calendar/create にアクセス、skill が追加できることを確認

- プルダウンの作成

```php:app/Http/Controllers/CalendarController.php
  ...
    public function create()
    {
        $skills = Skill::pluck('name', 'id'); // modelから読み込み、配列にする
        return view('calendar.create', compact('skills'));
    }

    public function store(Request $request)
    {
        $request->validate([
            'exp' => 'required|max:25'
        ]);
        Calendar::create([
            'user_id' => 2, //user2が存在している前提
            'skill_id' => $request->input('skill'),
            'exp' => $request->input('exp'),
            'created_at' => now(),
        ]);
        return redirect()->route('calendar.index')
            ->with('success', 'Post created successfully.');
    }
  ...
```

```php:resources/views/calendar/create.blade.php
        ...
        <h3>Add exp</h3>
        <form action="{{ route('calendar.store') }}" method="post">
          @csrf
          <div class="form-group">
            <label for="skill">skill</label>
            <select class="form-control" id="skill" name="skill">
              <option value="">選択してください</option>
              @foreach ($skills as $index => $name)
                  <option value="{{ $index }}">{{ $name }}</option>
              @endforeach
            </select>
          </div>
          <div class="form-group">
            <label for="exp">exp</label>
            <input type="text" class="form-control" id="exp" name="exp" required>
          </div>
          <button type="submit" class="btn btn-primary">add exp!</button>
        </form>
        ...
```

- プルダウンが作成される
  ![スクリーンショット 2024-12-15 10.24.55.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/e1a56b78-76f4-4155-5260-1697e71fb3ec.jpeg)

- validation の日本語化

```php:resources/lang/ja/validation.php
<?php

return [

  /*
    |--------------------------------------------------------------------------
    | Validation Language Lines
    |--------------------------------------------------------------------------
    |
    | The following language lines contain the default error messages used by
    | the validator class. Some of these rules have multiple versions such
    | as the size rules. Feel free to tweak each of these messages here.
    |
    */
  'attributes' => [
    'name' => '名前',
  ],

  'accepted'        => ':attributeを承認してください。',
  'active_url'      => ':attributeは、有効なURLではありません。',
  'after'           => ':attributeには、:dateより後の日付を指定してください。',
  'after_or_equal'  => ':attributeには、:date以降の日付を指定してください。',
  'alpha'           => ':attributeには、アルファベッドのみ使用できます。',
  'alpha_dash'      => ":attributeには、英数字('A-Z','a-z','0-9')とハイフンと下線('-','_')が使用できます。",
  'alpha_num'       => ":attributeには、英数字('A-Z','a-z','0-9')が使用できます。",
  'array'           => ':attributeには、配列を指定してください。',
  'before'          => ':attributeには、:dateより前の日付を指定してください。',
  'before_or_equal' => ':attributeには、:date以前の日付を指定してください。',
  'between'         => [
    'numeric' => ':attributeには、:minから、:maxまでの数字を指定してください。',
    'file'    => ':attributeには、:min KBから:max KBまでのサイズのファイルを指定してください。',
    'string'  => ':attributeは、:min文字から:max文字にしてください。',
    'array'   => ':attributeの項目は、:min個から:max個にしてください。',
  ],
  'boolean'              => ":attributeには、'true'か'false'を指定してください。",
  'confirmed'            => ':attributeと:attribute確認が一致しません。',
  'date'                 => ':attributeは、正しい日付ではありません。',
  'date_equals'          => ':attributeは:dateに等しい日付でなければなりません。',
  'date_format'          => ":attributeの形式は、':format'と合いません。",
  'different'            => ':attributeと:otherには、異なるものを指定してください。',
  'digits'               => ':attributeは、:digits桁にしてください。',
  'digits_between'       => ':attributeは、:min桁から:max桁にしてください。',
  'dimensions'           => ':attributeの画像サイズが無効です',
  'distinct'             => ':attributeの値が重複しています。',
  'email'                => ':attributeは、有効なメールアドレス形式で指定してください。',
  'ends_with'            => 'The :attribute must end with one of the following: :values',
  'exists'               => '選択された:attributeは、有効ではありません。',
  'file'                 => ':attributeはファイルでなければいけません。',
  'filled'               => ':attributeは必須です。',
  'gt'                   => [
    'numeric' => ':attributeは、:valueより大きくなければなりません。',
    'file'    => ':attributeは、:value KBより大きくなければなりません。',
    'string'  => ':attributeは、:value文字より大きくなければなりません。',
    'array'   => ':attributeの項目数は、:value個より大きくなければなりません。',
  ],
  'gte'                  => [
    'numeric' => ':attributeは、:value以上でなければなりません。',
    'file'    => ':attributeは、:value KB以上でなければなりません。',
    'string'  => ':attributeは、:value文字以上でなければなりません。',
    'array'   => ':attributeの項目数は、:value個以上でなければなりません。',
  ],
  'image'                => ':attributeには、画像を指定してください。',
  'in'                   => '選択された:attributeは、有効ではありません。',
  'in_array'             => ':attributeが:otherに存在しません。',
  'integer'              => ':attributeには、整数を指定してください。',
  'ip'                   => ':attributeには、有効なIPアドレスを指定してください。',
  'ipv4'                 => ':attributeはIPv4アドレスを指定してください。',
  'ipv6'                 => ':attributeはIPv6アドレスを指定してください。',
  'json'                 => ':attributeには、有効なJSON文字列を指定してください。',
  'lt'                   => [
    'numeric' => ':attributeは、:valueより小さくなければなりません。',
    'file'    => ':attributeは、:value KBより小さくなければなりません。',
    'string'  => ':attributeは、:value文字より小さくなければなりません。',
    'array'   => ':attributeの項目数は、:value個より小さくなければなりません。',
  ],
  'lte'                  => [
    'numeric' => ':attributeは、:value以下でなければなりません。',
    'file'    => ':attributeは、:value KB以下でなければなりません。',
    'string'  => ':attributeは、:value文字以下でなければなりません。',
    'array'   => ':attributeの項目数は、:value個以下でなければなりません。',
  ],
  'max'                  => [
    'numeric' => ':attributeには、:max以下の数字を指定してください。',
    'file'    => ':attributeには、:max KB以下のファイルを指定してください。',
    'string'  => ':attributeは、:max文字以下にしてください。',
    'array'   => ':attributeの項目は、:max個以下にしてください。',
  ],
  'mimes'                => ':attributeには、:valuesタイプのファイルを指定してください。',
  'mimetypes'            => ':attributeには、:valuesタイプのファイルを指定してください。',
  'min'                  => [
    'numeric' => ':attributeには、:min以上の数字を指定してください。',
    'file'    => ':attributeには、:min KB以上のファイルを指定してください。',
    'string'  => ':attributeは、:min文字以上にしてください。',
    'array'   => ':attributeの項目は、:min個以上にしてください。',
  ],
  'not_in'               => '選択された:attributeは、有効ではありません。',
  'not_regex'            => ':attributeの形式が無効です。',
  'numeric'              => ':attributeには、数字を指定してください。',
  'password'             => ':attributeが間違っています',
  'present'              => ':attributeが存在している必要があります。',
  'regex'                => ':attributeには、有効な正規表現を指定してください。',
  'required'             => ':attributeは、必ず指定してください。',
  'required_if'          => ':otherが:valueの場合、:attributeを指定してください。',
  'required_unless'      => ':otherが:values以外の場合、:attributeを指定してください。',
  'required_with'        => ':valuesが指定されている場合、:attributeも指定してください。',
  'required_with_all'    => ':valuesが全て指定されている場合、:attributeも指定してください。',
  'required_without'     => ':valuesが指定されていない場合、:attributeを指定してください。',
  'required_without_all' => ':valuesが全て指定されていない場合、:attributeを指定してください。',
  'same'                 => ':attributeと:otherが一致しません。',
  'size'                 => [
    'numeric' => ':attributeには、:sizeを指定してください。',
    'file'    => ':attributeには、:size KBのファイルを指定してください。',
    'string'  => ':attributeは、:size文字にしてください。',
    'array'   => ':attributeの項目は、:size個にしてください。',
  ],
  'starts_with'          => ':attributeは、次のいずれかで始まる必要があります。:values',
  'string'               => ':attributeには、文字を指定してください。',
  'timezone'             => ':attributeには、有効なタイムゾーンを指定してください。',
  'unique'               => '指定の:attributeは既に使用されています。',
  'uploaded'             => ':attributeのアップロードに失敗しました。',
  'url'                  => ':attributeは、有効なURL形式で指定してください。',
  'uuid'                 => ':attributeは、有効なUUIDでなければなりません。',

  /*
    |--------------------------------------------------------------------------
    | Custom Validation Language Lines
    |--------------------------------------------------------------------------
    |
    | Here you may specify custom validation messages for attributes using the
    | convention "attribute.rule" to name the lines. This makes it quick to
    | specify a specific custom language line for a given attribute rule.
    |
    */

  'custom' => [
    'attribute-name' => [
      'rule-name' => 'custom-message',
    ],
  ],

  /*
    |--------------------------------------------------------------------------
    | Custom Validation Attributes
    |--------------------------------------------------------------------------
    |
    | The following language lines are used to swap our attribute placeholder
    | with something more reader friendly such as "E-Mail Address" instead
    | of "email". This simply helps us make our message more expressive.
    |
    */
];
```

## 今思った

- データベースの skill はユーザー id をつけていない。
- よって他のユーザーの skill も見れるけど、プルダウンにすると多すぎる
- プルダウンを辞める or 自分のスキルだけに変更したい

## ユーザー管理

- 別記事にする予定

## 参考にした記事

- session について

https://qiita.com/yutaka_pg/items/f0103c3171b75146c28a

- Laravel11 でのユーザー認証

https://pointsandlines.jp/server-side/php/laravel-11-auth-self-made-2

- テーブルの接続

https://zenn.dev/aono/articles/e73c6c3f6be836

- laravel の crud

https://kinsta.com/jp/blog/laravel-crud/

- validation の日本語化

https://qiita.com/dam-san/items/3c0dc498d2fcab0bda0d

- プルダウンの作り方

https://qiita.com/yastinbieber/items/26afd85afc146bc62fcb
