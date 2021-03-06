git bc551e44eaed92ca1bbe8292bb4680e65d6f7097 

---

# Отношения Eloquent:

- [Вступление](#introduction)
- [Определение отношений](#defining-relationships)
    - [Один к одному](#one-to-one)
    - [Один ко многим](#one-to-many)
    - [Многие ко многим](#many-to-many)
    - [Многие ко многим через посредника (Has Many Through)](#has-many-through)
    - [Полиморфические отношения](#polymorphic-relations)
    - [Полиморфические отношения многие ко многим](#many-to-many-polymorphic-relations)
- [Запросы к отношениям](#querying-relations)
    - [Жадная загрузка (eager loading)](#eager-loading)
    - [Ограничение жадной загрузки](#constraining-eager-loads)
    - [Отложенная загрузка](#lazy-eager-loading)
- [Вставка связанных моделей](#inserting-related-models)
    - [Многие ко многим](#inserting-many-to-many-relationships)
    - [Обновление родительских таймстемпов](#touching-parent-timestamps)

<a name="introduction"></a>
## Вступление

Ваши таблицы скорее всего как-то связаны с другими таблицами БД. Например, статья в блоге может иметь несколько комментариев, а заказ может быть с связан с оставившим его пользователем. Eloquent упрощает работу с такими отношениями. Laravel поддерживает несколько типов связей: 

    - [Один к одному](#one-to-one)
    - [Один ко многим](#one-to-many)
    - [Многие ко многим](#many-to-many)
    - [Has Many Through](#has-many-through)
    - [Полиморфические отношения](#polymorphic-relations)
    - [Полиморфические отношения многие ко многим](#many-to-many-polymorphic-relations)

<a name="defining-relationships"></a>
##  Определение отношений

Отношения Eloquent определяются при помощи методов в модели Eloquent. Т.к. связи (как и сами модели) по сути являются [конструкторами запросов](/docs/{{version}}/queries), определение связей в виде методов позволяет использовать мощный механизм сцепления методов в цепочку и построения запроса. Например:

    $user->posts()->where('active', 1)->get();

Но, прежде чем мы погрузимся глубоко в использование связей, рассмотрим как определять каждый из типов:

<a name="one-to-one"></a>
### Один к одному

Связь вида «один к одному» является очень простой. К примеру, модель `User` может иметь один `Phone`. Для определения такой связи мы заведем метод `phone` в модели `User`. Метод `phone` должен вернуть результат метода `hasOne` базового класса Eloquent модели:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Получить телефон, связанный с пользователем.
         */
        public function phone()
        {
            return $this->hasOne('App\Phone');
        }
    }

Первый аргумент, передаваемый в метод `hasOne` - имя модели связи. Теперь, когда связь определена можно получить связанную запись при помощи динамического атрибута. Динамические атрибуты позволяют обращаться к отношениям так, будто они являются атрибутами самой модели:

    $phone = User::find(1)->phone;

Eloquent по умолчанию предугадывает имя внешнего ключа по имени модели. В данном случае подразумевается что модель `Phone` имеет внешний ключ `user_id`. Если хотите переопределить это правило, имя ключа можно передать вторым параметром в метод`hasOne`:

    return $this->hasOne('App\Phone', 'foreign_key');

Также Eloquent предполагает, что значение внешнего ключа будет равно `id` (или атрибуту, указанному в `$primaryKey`) родительской модели. Другими словами, Eloquent будет искать пользователя с `id` равным столбцу `user_id` в модели `Phone`. Если хотите использовать другой атрибут (не `id`) в качестве идентификатора связи, можно передать третий параметр в метод `hasOne` указав свой ключ:

    return $this->hasOne('App\Phone', 'foreign_key', 'local_key');

#### Определение обратного отношения

Итак мы можем получить модель телефона (`Phone`) из модели пользователя (`User`). Давайте теперь определим связь на стороне телефона, что позволит нам иметь доступ к модели пользователя (`User`), владельцу телефона. Мы можем определить обратное отношение к `hasOne` при помощи метода `belongsTo`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Phone extends Model
    {
        /**
         * Получить пользователя - владельца телефона
         */
        public function user()
        {
            return $this->belongsTo('App\User');
        }
    }

В примере выше Eloquent попытается сопоставить `user_id` модели `Phone` атрибуту `id` модели `User`. Eloquent определяет дефолтное имя внешнего ключа по имени метода, который задает отношение, добавив к нему суффикс `_id`. Однако, если имя внешнего ключа в модели `Phone` не `user_id`, можно передать собственное имя ключа вторым параметром метода `belongsTo`:

    /**
     * Получить пользователя - владельца телефон
     */
    public function user()
    {
        return $this->belongsTo('App\User', 'foreign_key');
    }

Если родительская модель не использует `id` в качестве первичного ключа, или вы хотите связать дочернею модель с родительской по другой колонке, можно передать третьим параметром в метод `belongsTo` имя ключа для связи:

    /**
     * Получить пользователя - владельца телефон
     */
    public function user()
    {
        return $this->belongsTo('App\User', 'foreign_key', 'other_key');
    }

<a name="one-to-many"></a>
### Один ко многим

Тип связи "один ко многим" используется для определения таких отношений, в которых одна модель может иметь  неограниченное количество других моделей. Например статья в блоге может иметь неограниченное количество комментариев. Как и любые другие отношения Eloquent, связь "один ко многим" определяется при помощи метода модели Eloquent:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Post extends Model
    {
        /**
         * Получить комментарии к записи.
         */
        public function comments()
        {
            return $this->hasMany('App\Comment');
        }
    }

Не забывайте, что Eloquent будет автоматически определять внешний ключ в модели `Comment`. Обычно при этом берется имя родительской модели в "змеином_регистре" ("snake case") и добавляется суффикс `_id`. Например, в нашем случае в модели `Comment` Eloquent будет искать поле `post_id`.

Когда отношение описано, мы можем получить коллекцию комментариев через атрибут `comments`. Помните, что динамические атрибуты позволяют обращаться к отношениям так, будто они являются атрибутами самой модели:

    $comments = App\Post::find(1)->comments;

    foreach ($comments as $comment) {
        //
    }

И конечно, т.к. наши отношения являются по сути построителями запроса, вы можете строить цепь вызовов добавляя необходимые условия, после вызова метода `comments`:

    $comments = App\Post::find(1)->comments()->where('title', 'foo')->first();

Как и в случае с методом `hasOne`, можно указать собственные имена ключей в таблицах, передавая их дополнительными параметрами в `hasMany`:

    return $this->hasMany('App\Comment', 'foreign_key');

    return $this->hasMany('App\Comment', 'foreign_key', 'local_key');

#### Определение обратного отношения

Теперь, когда у нас есть метод  для получения всех комментариев записи блога, давайте определим отношение для получения родительской записи из модели комментария. Для описания обратной связи `hasMany` служит метод `belongsTo`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Comment extends Model
    {
        /**
         * Получить родительскую запись комментария.
         */
        public function post()
        {
            return $this->belongsTo('App\Post');
        }
    }

Теперь, когда отношение описано, мы можем получить родительский объект `Post` для `Comment` через "динамический атрибут" `post`:

    $comment = App\Comment::find(1);

    echo $comment->post->title;

В примере выше, Eloquent будет пытаться найти объект `Post` с `id` равным `post_id` модели `Comment`. Eloquent по умолчанию определяет имя внешнего ключа по имени метода, задающего отношение с суффиксом `_id`. Однако, если в модели `Comment` внешний ключ не равен `post_id`, можно передать свое имя  в метод `belongsTo`:

    /**
     * Получить родительскую запись комментария.
     */
    public function post()
    {
        return $this->belongsTo('App\Post', 'foreign_key');
    }

Если родительская модель не использует `id` в качестве первичного ключа, или вы хотите связать дочернею модель с родительской по другой колонке, можно передать третьим параметром в метод `belongsTo` имя ключа для связи:

    /**
     * Получить родительскую запись комментария.
     */
    public function post()
    {
        return $this->belongsTo('App\Post', 'foreign_key', 'other_key');
    }

<a name="many-to-many"></a>
### Многие ко многим

Отношения типа «многие ко многим» - более сложные, чем остальные виды отношений. Примером может служить пользователь, имеющий много ролей, где роли также относятся ко многим пользователям. Например, один пользователь может иметь роль «Admin». Для этой связи нужны три таблицы : `users`, `roles` и `role_user`. Название таблицы `role_user` происходит от **упорядоченного по алфавиту** имён связанных моделей и она должна иметь поля `user_id` и `role_id`.

Вы можете определить отношение «многие ко многим» через метод `belongsToMany`. Давайте для примера определим связь `roles` в модели `User`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Роли, к которым принадлежит пользователь.
         */
        public function roles()
        {
            return $this->belongsToMany('App\Role');
        }
    }

Когда отношение описано, мы можем получить роли пользователя через динамический атрибут `roles`:

    $user = App\User::find(1);

    foreach ($user->roles as $role) {
        //
    }

И конечно же, как и в случае с любым другим типом отношений, можно выстраивать цепочки вызовов таким образом:

    $roles = App\User::find(1)->roles()->orderBy('name')->get();

Как упомянуто выше, имя связующей таблицы по умолчанию строится по именам моделей в алфавитном порядке. Однако его можно переопределить. Делается это при помощи второго параметра метода `belongsToMany`:

    return $this->belongsToMany('App\Role', 'role_user');

Помимо собственного названия связующей таблицы​, можно также переопределить имена колонок-ключей при помощи дополнительных параметров метода `belongsToMany`. Третий аргумент — колонка, ссылающаяся на модель, в которой вы описываете отношение, четвертый — колонка, ссылающаяся на модель с которой строится связь:

    return $this->belongsToMany('App\Role', 'role_user', 'user_id', 'role_id');

#### Определение обратного отношения

Для определения обратного отношения «многие ко многим» необходимо просто добавить такой же вызов метода `belongsToMany` но, со стороны другой модели. В продолжение нашего примера с ролями пользователя,  определим метод `users` в модели `Role`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Role extends Model
    {
        /**
         * Пользователи, которые принадлежат данной роли.
         */
        public function users()
        {
            return $this->belongsToMany('App\User');
        }
    }

Как видите, отношение описывается точно также, как обратное, со стороны пользователя, с той лишь разницей, что ссылаемся мы теперь на модель `App\User`. Т.к. вызов метода `belongsToMany` является таким же точно, как и выше, все опции по заданию собственных имен таблиц и колонок выглядят идентично.

#### Работа с данными связующих таблиц

Как вы уже знаете отношения типа «многие ко многим» требует дополнительную связующую таблицу. Eloquent  позволяет работать с этой таблицей, что бывает весьма полезно. Предположим, что наш объект `User` имеет много ролей (объектов `Role`). После того, как мы получили объект отношения, мы можем получить доступ к связующей таблице при помощи атрибута `pivot` у каждого из объектов:

    $user = App\User::find(1);

    foreach ($user->roles as $role) {
        echo $role->pivot->created_at;
    }

Обратите внимание, что у каждой из моделей `Role` есть автоматически созданный атрибут `pivot`. Этот атрибут представляет из себя модель с данными связующей таблицы, и может использоваться как обычный объект Eloquent.

По умолчанию, в объекте `pivot` будут присутствовать только ключи  моделей. Если ваша связующая таблица содержит дополнительные атрибуты, их необходимо перечислить при описании отношения:

    return $this->belongsToMany('App\Role')->withPivot('column1', 'column2');

Если вы хотите, чтобы связующая таблица автоматически поддерживала таймстемпы `created_at` и `updated_at`, используйте метод `withTimestamps` при описании отношения:

    return $this->belongsToMany('App\Role')->withTimestamps();

#### Фильтрация условий при помощи связующей таблицы

Вы можете фильтровать результаты, возвращенные отношением `belongsToMany` при помощи условий к связующей таблице:

    return $this->belongsToMany('App\Role')->wherePivot('approved', 1);

    return $this->belongsToMany('App\Role')->wherePivotIn('approved', [1, 2]);


<a name="has-many-through"></a>
### Многие ко многим через посредника (Has Many Through)

Отношение "многие ко многим через посредника" предоставляет удобный способ для доступа к отдаленным отношениям через отношение-посредник. Например, Страна (`Country`) может иметь много блог-записей(`Post`) через модель пользователя (`User`). В примере ниже показано, как легко можно получить все блог-посты для указанной страны. Давайте посмотрим на таблицы, необходимые для построения такой связи:

    countries
        id - integer
        name - string

    users
        id - integer
        country_id - integer
        name - string

    posts
        id - integer
        user_id - integer
        title - string

Хотя таблица `posts` не содержит поле `country_id`, отношение `hasManyThrough` позволяет получить доступ к блог-постам, относящимся к стране через атрибут `$country->posts`. Чтобы сделать такую выборку Eloquent использует поле `country_id` на связующей таблице `users`. После получения ID пользователей, они используются для выборки по таблице `posts`.

Теперь, когда ясна структура таблиц для этого отношения, давайте опишем его в модели `Country`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Country extends Model
    {
        /**
         * Получить все посты  для страны.
         */
        public function posts()
        {
            return $this->hasManyThrough('App\Post', 'App\User');
        }
    }

Первый аргумент метода `hasManyThrough` является именем конечной модели, второй аргумент — модель-посредник.

Имена внешних ключей по умолчанию будут строиться как обычно принято в Eloquent. Если вы хотите использовать свои имена ключей, их можно передать следующими параметрами метода `hasManyThrough`. Третий параметр — имя ключа в модели-посреднике, четвертый — имя ключа конечной модели, пятый — местный ключ, в данной модели.

    class Country extends Model
    {
        public function posts()
        {
            return $this->hasManyThrough(
                'App\Post', 'App\User', 
                'country_id', 'user_id', 'id');
        }
    }

<a name="polymorphic-relations"></a>
### Полиморфические отношения

#### Структура таблиц

Полиморфические отношения позволяют модели быть связанной с более, чем одной моделью. Например, пользователи приложений могут "лайкать" как посты так и комментарии к ним. Используя полиморфическую связь, вы можете использовать одну таблицу `likes` для обоих сценариев. Давайте для начала посмотрим на структуру таблиц, которая необходима для построения подобного отношения:

    posts
        id - integer
        title - string
        body - text

    comments
        id - integer
        post_id - integer
        body - text

    likes
        id - integer
        likeable_id - integer
        likeable_type - string

Два важных поля, на которые стоит обратить внимание - `likeable_id` и `likeable_type` в таблице `likes`. Поле `likeable_id` будет хранить ID поста или комментария, а поле `likeable_type` содержать имя класса соответствующей модели. Именно по значению поля `likeable_type` ORM  определит какой "тип" модели вернуть при обращении к связи `likeable`.

#### Структура модели

Теперь давайте рассмотрим определение такого рода отношений в модели:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Like extends Model
    {
        /**
         * Получать все модели с лайками.
         */
        public function likeable()
        {
            return $this->morphTo();
        }
    }

    class Post extends Model
    {
        /**
         * Получить все лайки поста.
         */
        public function likes()
        {
            return $this->morphMany('App\Like', 'likeable');
        }
    }

    class Comment extends Model
    {
        /**
         * Получить все лайки комментария.
         */
        public function likes()
        {
            return $this->morphMany('App\Like', 'likeable');
        }
    }

#### Доступ к данным полиморфических отношений

У нас есть структура таблиц и модели, теперь можно обращаться к отношениям из моделей. Например, получить лайки поста можно при помощи динамического атрибута `likes`:

    $post = App\Post::find(1);

    foreach ($post->likes as $like) {
        //
    }

Также можно получить владельца полиморфической модели при помощи метода возвращающего вызов `morphTo`. В нашем случае, это метод `likeable` в модели `Like`. Можно использовать динамический атрибут:

    $like = App\Like::find(1);

    $likeable = $like->likeable;

Отношение `likeable` в модели `Like` вернет либо инстанс либо модели `Post` либо `Comment`, в зависимости от принадлежности «лайка».

#### Переопределение полиморфических типов

По умолчанию, Laravel использует полное имя модели в качестве типа модели. Например, в нашем примере, где `Like` может принадлежать как  модели `Post` так и `Comment`, значение `likable_type` по умолчанию будет либо `App\Post` либо `App\Comment` соответственно. Однако, возможно вы захотите отвязать данные базы от структуры приложения. В этом случае можно определить «карту превращений» ("morph map") чтобы дать инструкции Eloquent по превращению данных таблицы в модели:

    use Illuminate\Database\Eloquent\Relations\Relation;

    Relation::morphMap([
        App\Post::class,
        App\Comment::class,
    ]);

Или можно задать строку, которая будет являться ключом модели:

    use Illuminate\Database\Eloquent\Relations\Relation;

    Relation::morphMap([
        'posts' => App\Post::class,
        'likes' => App\Like::class,
    ]);

«Карту превращений» (`morphMap`) можно зарегистрировать в `AppServiceProvider` или создать для этой цели отдельный сервис-провайдер.

<a name="many-to-many-polymorphic-relations"></a>
### Полиморфические отношения «многие ко многим»

#### Структура таблиц

Помимо стандартных полиморфических отношений, вы можете определить полиморфические отношения «многие ко многим». Например, у нас есть блог, в котором могут публиковаться `Post` и `Video`, и каждый из них может иметь набор тэгов `Tag`. Полиморфические отношения «многие ко многим» ползволят в таком случае иметь один список тегов для постов и видео. Для начала, рассмотрим структуру таблиц:

    posts
        id - integer
        name - string

    videos
        id - integer
        name - string

    tags
        id - integer
        name - string

    taggables
        tag_id - integer
        taggable_id - integer
        taggable_type - string

#### Структура модели

Затем опишем отношения в моделях. У классов `Post` и `Video` будет метод `tags`, который вызывает метод `morphToMany` базового класса Eloquent:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Post extends Model
    {
        /**
         * Получить все теги поста.
         */
        public function tags()
        {
            return $this->morphToMany('App\Tag', 'taggable');
        }
    }

#### Определение обратного отношения

Затем, в модели `Tag` необходимо задать метод для получения связанных моделей. В нашем случае определим методы `posts` и `videos`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Tag extends Model
    {
        /**
         * Получить все посты, связанные с тегом.
         */
        public function posts()
        {
            return $this->morphedByMany('App\Post', 'taggable');
        }

        /**
         * Получить все видео, связанные с тегом.
         */
        public function videos()
        {
            return $this->morphedByMany('App\Video', 'taggable');
        }
    }

#### Доступ к данным отношения

После того, как база данных и модели определены, можно получить данные отношений  из моделей. Например для получения всех тегов поста просто используйте динамический атрибут `tags`:

    $post = App\Post::find(1);

    foreach ($post->tags as $tag) {
        //
    }

Также можно получить владельца полиморфической модели при помощи метода возвращающего вызов `morphedByMany`. В нашем случае, это метод `posts` или `videos` в модели `Tag`. Можно использовать динамический атрибут:

    $tag = App\Tag::find(1);

    foreach ($tag->videos as $video) {
        //
    }

<a name="querying-relations"></a>
## Запросы к отношениям

Так как все отношения Eloquent определяются в виде функций, вы можете получить инстанс отношения без исполнения запроса на получение данных. К тому же, все типы отношений Eloquent являются [построителями запроса](/docs/{{version}}/queries), что позволяет добавлять условия по цепочке вызовов до того, как произойдет сам SQL запрос.

Рассмотрим пример: представьте некий блог, в котором одной модели `User` может соответствовать много  моделей `Post`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Получить все посты пользователя.
         */
        public function posts()
        {
            return $this->hasMany('App\Post');
        }
    }

Вы можете составить запрос к отношению `posts`, добавив дополнительные условия примерно так:

    $user = App\User::find(1);

    $user->posts()->where('active', 1)->get();

Обратите внимание, что при запросе к отношению можно использовать любой из [методов построения запроса](/docs/{{version}}/queries)!

#### Динамический атрибут или вызов метода

Если у вас нет необходимости добавлять дополнительные условия к получению данных отношений Eloquent читать отношение можно просто через динамический атрибут. Например, в нашем примере с пользователем `User` и постами `Post` мы можем получить все посты пользователя так:

    $user = App\User::find(1);

    foreach ($user->posts as $post) {
        //
    }

Динамические атрибуты являются атрибутами «отложенной загрузки» ("lazy loading"), это означает, что данные будут загружаться только при непосредственном обращении к атрибуту. Из-за этого разработчики часто используют [жадную загрузку (eager loading)](#eager-loading) чтобы предварительно получить данные отношений, которые точно будут использоваться при обращении к текущей модели. «Жадная загрузка» позволяет существенно снизить количество SQL запросов, необходимых для получения отношений модели.

#### Проверка существования отношения

При чтении записей модели, бывает необходимо ограничить результаты выборки на основании факта существования данных отношения. Например, надо выбрать все посты у которых есть хотя бы один комментарий. Для этой цели существует служит метод `has`, в который надо передать имя отношения:

    // Получить все посты у которых есть хоть один комментарий...
    $posts = App\Post::has('comments')->get();

Также можно добавить оператор и указать число:

    // Получить все посты у которых три и более комментариев...
    $posts = Post::has('comments', '>=', 3)->get();

Можно организовать вложенность через «точку». Например, получить все посты у которых есть хотя бы один комментарий с голосом так:

    // получить все посты у которых есть хотя бы один комментарий с голосом...
    $posts = Post::has('comments.votes')->get();

Для более сложных ситуаций пригодятся методы `whereHas` и `orWhereHas`, они служат для добавления условий "where" в запрос `has`. Эти методы позволяют добавить кастомизированные условия в выборку данных отношений, например добавить условие по содержанию комментария:

    // Получить все посты у которых есть хоть один комментарий с содержанием foo%
    $posts = Post::whereHas('comments', function ($query) {
        $query->where('content', 'like', 'foo%');
    })->get();

#### Подсчет результатов

Если вы хотите узнать число результатов, которое вернет отношение, воспользуйтесь методом `withCount`. Результат можно прочитать в свойстве модели `{relation}_count`:  

    $posts = App\Post::withCount('comments')->get();

    foreach ($posts as $post) {
        echo $post->comments_count;
    }

Вы можете подсчитать число результатов в нескольких отношениях:

    $posts = Post::withCount(['votes', 'comments' => function ($query) {
        $query->where('content', 'like', 'foo%');
    }])->get();

    echo $posts[0]->votes_count;
    echo $posts[0]->comments_count;


<a name="eager-loading"></a>
### Жадная загрузка

Динамические атрибуты отношений являются атрибутами «отложенной загрузки» ("lazy loading"), это означает, что данные будут загружаться только при непосредственном обращении к атрибуту. Однако, Eloquent дает возможность «жадной загрузки» ("eager load") отношений во время получения данных самой модели. Жадная загрузка снимает проблему N + 1 запросов. Для того чтобы понять, что такое проблема N + 1 запросов, рассмотрим модель `Book` связанную с моделью `Author`:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Book extends Model
    {
        /**
         * Получить автора книги.
         */
        public function author()
        {
            return $this->belongsTo('App\Author');
        }
    }

Теперь давайте получим все книги вместе с авторами:

    $books = App\Book::all();

    foreach ($books as $book) {
        echo $book->author->name;
    }

Этот цикл произведет один запрос к таблице с книгами, затем еще по запросу на каждую книгу для получения автора. И, если у нас 25 книг, цикл сделает 26 запросов: 1 для книг, и 25 для получения автора каждой книги.

К счастью, у нас есть «жадная загрузка», которая сведет все к 2 запросам. Какое отношение загрузить «жадно» можно указать при помощи метода `with`:

    $books = App\Book::with('author')->get();

    foreach ($books as $book) {
        echo $book->author->name;
    }

Для данной операции выполнится всего два запроса:

    select * from books

    select * from authors where id in (1, 2, 3, 4, 5, ...)

#### Жадная загрузка нескольких отношений

В некоторых ситуациях может понадобиться одновременная жадная загрузка сразу нескольких отношений. Для этого просто перечислите названия отношений в качестве аргументов метода `with`:

    $books = App\Book::with('author', 'publisher')->get();

#### Вложенная жадная загрузка

Для жадной загрузки вложенных отношений используйте синтаксис с "точкой". Для примера давайте загрузим книги с авторами и их персональными данными одним выражением Eloquent:

    $books = App\Book::with('author.contacts')->get();

<a name="constraining-eager-loads"></a>
### Условия при жадной загрузке

Иногда может понадобиться жадная загрузка отношения, но с дополнительным условием на выборку связанных данных Вот пример:

    $users = App\User::with(['posts' => function ($query) {
        $query->where('title', 'like', '%first%');

    }])->get();

В этом примере Eloquent загрузит только те посты пользователя, `title` которых содержит слово `first`. И конечно вы можете использовать любые другие методы [построения запросов](/docs/{{version}}/queries) для формирования своих условий:

    $users = App\User::with(['posts' => function ($query) {
        $query->orderBy('created_at', 'desc');

    }])->get();

<a name="lazy-eager-loading"></a>
### Отложенная жадная загрузка

Бывают ситуации, когда необходимо жадно загрузить отношения уже после того как получили данные родительской модели. К примеру, если решение о загрузке отношений принимается динамически:

    $books = App\Book::all();

    if ($someCondition) {
        $books->load('author', 'publisher');
    }

Если вам необходимо добавить свои условия на выборку отношений, передайте `Closure` в метод `load`:

    $books->load(['author' => function ($query) {
        $query->orderBy('published_date', 'asc');
    }]);

<a name="inserting-related-models"></a>
## Добавление связанных моделей

#### Метод Save (сохранение)

Eloquent предоставляет удобные метода для добавления новых моделей в отношения. Например, нам надо добавить новый комментарий (`Comment`) к посту (`Post`). Вместо того, чтобы вручную указывать атрибут `post_id` у модели `Comment`, вы можете создать `Comment` из метода `save` самого отношения:

    $comment = new App\Comment(['message' => 'A new comment.']);

    $post = App\Post::find(1);

    $post->comments()->save($comment);

Обратите внимание, мы не обращаемся к динамическому атрибуту `comments`, а используем метод `comments()` для получения инстанса отношения. Метод `save` автоматически проставит нужное значение атрибуту `post_id` в новой модели `Comment`.

Если необходимо добавить несколько связанных моделей, используйте метод `saveMany`:

    $post = App\Post::find(1);

    $post->comments()->saveMany([
        new App\Comment(['message' => 'A new comment.']),
        new App\Comment(['message' => 'Another comment.']),
    ]);

#### Метод Save и отношение «многие ко многим»

При работе с отношениям типа «многие ко многим» метод `save` принимает массив дополнительных атрибутов связующей таблицы в качестве второго параметра:

    App\User::find(1)->roles()->save($role, ['expires' => $expires]);

#### Метод Create (создание)

Наряду с методами `save` и `saveMany`, существует метод `create`, который принимает на вход массив атрибутов, создает модель и сохраняет ее в БД. Повторим еще раз, разница между `save` и `create` состоит в том, что `save` принимает уже готовую модель Eloquent, а `create` - простой PHP массив:

    $post = App\Post::find(1);

    $comment = $post->comments()->create([
        'message' => 'A new comment.',
    ]);

Перед использованием метода `create`, ознакомьтесь с документацией по массовому [присвоению атрибутов](/docs/{{version}}/eloquent#mass-assignment).

<a name="updating-belongs-to-relationships"></a>
#### Обновление отношения "Belongs To"

При обновлении отношения типа `belongsTo` можно использовать метод `associate`. Этот метод установит значение внешнего ключа на дочерней модели:

    $account = App\Account::find(10);

    $user->account()->associate($account);

    $user->save();

Для удаления связи `belongsTo`, можно использовать метод `dissociate`. Этот метод обнуляет как внешний ключ  так и ссылку на дочерней модели:

    $user->account()->dissociate();

    $user->save();

<a name="inserting-many-to-many-relationships"></a>
### Отношения многие-ко-многим

#### Связывание / Отвязывание

Для удобства работы с отношениями типа «многие-ко-многим» Eloquent предоставляет несколько дополнительных методов. Давайте предположим, что у нас есть пользователь, который имеет несколько ролей, роль в свою очередь может относиться ко многим пользователям. Для того, чтобы связать роль с пользователем путем добавления записи в связующую таблицу, используйте метод `attach`:

    $user = App\User::find(1);

    $user->roles()->attach($roleId);

При связывании можно также передать массив с дополнительными данными для  связующей таблицы:

    $user->roles()->attach($roleId, ['expires' => $expires]);

Для того, чтобы отвязать роль от пользователя, удалить запись из связующей таблицы, используйте метод `detach`. Метод `detach` удалит только запись из связубщей таблица, сами же модели останутся без изменений:

    // Отвязать роль от пользователя...
    $user->roles()->detach($roleId);

    // Отвязать все роли от пользователя...
    $user->roles()->detach();

В целях удобства, методы `attach` и `detach` также принимают на вход массивы ID:

    $user = App\User::find(1);

    $user->roles()->detach([1, 2, 3]);

    $user->roles()->attach([1 => ['expires' => $expires], 2, 3]);

#### Обновление записи связующей таблицы

Если необходимо обновить данные существующей записи связующей таблицы, используйте метод `updateExistingPivot`:

    $user = App\User::find(1);

	$user->roles()->updateExistingPivot($roleId, $attributes);

#### Синхронизация

Посмотрите также в сторону метода `sync` для создания связей «многие ко многим». Метод `sync` принимает массив ID для связующей таблицы. Те ID которые отсутствуют в масиве будут удалены из таблицы связей.  Таким образом в результате выполнения метода записи в связующей таблице будут соответствовать массиву, который в него передали:

    $user->roles()->sync([1, 2, 3]);

Вместе с ID можно передать также дополнительные данные для связующей таблицы:

    $user->roles()->sync([1 => ['expires' => true], 2, 3]);

<a name="touching-parent-timestamps"></a>
###  Обновление родительских таймстемпов

Когда модель связана с другой моделью через отношения типа `belongsTo` или  `belongsToMany`, как например комментарий (`Comment`) принадлежит посту (`Post`),  бывает полезно обновить таймстемп родителя при обновлении дочерней модели. Например, при обновлении комментария мы хотим освежить атрибут `updated_at` родительской модели `Post`. С Eloquent это просто. Просто добавьте в дочернюю модель атрибут `touches` с перечислением имен отношений:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Comment extends Model
    {
        /**
         * Перечень отношений, которые необходимо освежить.
         *
         * @var array
         */
        protected $touches = ['post'];

        /**
         * Получить пост, к которому принадлжеит комментарий.
         */
        public function post()
        {
            return $this->belongsTo('App\Post');
        }
    }

Теперь, при обновлении записи `Comment`, родительский `Post` также «освежит» атрибут `updated_at`:

    $comment = App\Comment::find(1);

    $comment->text = 'Edit to this comment!';

    $comment->save();
 
