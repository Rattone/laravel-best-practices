# Laravel Best Practices

## Contenuto

[Principio di singola responsabilità](#principio-di-singola-responsabilità)

[Modelli grossi, controller snelli](#modelli-grossi-controller-snelli)

[Validazione](#validazione)

[La logica di business dovrebbe stare nelle classi di servizio](#la-logica-di-business-dovrebbe-stare-nelle-classi-di-servizio)

[Non ripeterti (DRY: Don't Repeat Yourself)](#non-ripeterti-dry-dont-repeat-yourself)

[Favorisci l'utilizzo dei Model Eloquent rispetto al Query Builder e alle query SQL raw. Preferisci le Collection agli array](#favorisci-lutilizzo-dei-model-eloquent-rispetto-al-query-builder-e-alle-query-sql-raw-preferisci-le-collection-agli-array)

[Assegnazione di massa](#assegnazione-di-massa)

[Non eseguire query nei template Blade e utilizza l'eager loading (problema N + 1)](#non-eseguire-query-nei-template-blade-e-utilizzare-leager-loading-problema-n--1)

[Commenta il tuo codice, ma cerca anche di rendere autoesplicativi i nomi di metodi e variabili](#commenta-il-tuo-codice-ma-cerca-anche-di-rendere-autoesplicativi-i-nomi-di-metodi-e-variabili)

[Non inserire JS e CSS nei template Blade e non inserire HTML nelle classi PHP](#non-inserire-js-e-css-nei-template-blade-e-non-inserire-html-nelle-classi-php)

[Usa file di configurazione e lingua, costanti anziché testo nel codice](#usa-file-di-configurazione-e-lingua-costanti-anziché-testo-nel-codice)

[Utilizza gli strumenti standard Laravel accettati dalla community](#use-standard-laravel-tools-accepted-by-community)

[Segui le convenzioni di denominazione di Laravel](#follow-laravel-naming-conventions)

[Utilizza la sintassi più breve e più leggibile ove possibile](#use-shorter-and-more-readable-syntax-where-possible)

[Utilizza il contenitore o le facciate IoC anziché la nuova classe](#use-ioc-container-or-facades-instead-of-new-class)

[Non prelevare direttamente i dati dal file `.env`](#do-not-get-data-from-the-env-file-directly)

[Memorizza le date nel formato standard. Utilizzare accessori e mutatori per modificare il formato della data](#store-dates-in-the-standard-format-use-accessors-and-mutators-to-modify-date-format)

[Altre buone pratiche](#other-good-practices)

### Principio di singola responsabilità

Una classe e un metodo dovrebbero avere una sola responsabilità.

Male:

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Buono:

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[🔝Torna ai contenuti](#contenuto)

### Modelli grossi, controller snelli

Inserisci tutta la logica legata al DB nei Model Eloquent oppure nei Repository a seconda che tu stia usando il Query Builder o le query SQL raw.

Male:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Buono:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝Torna ai contenuti](#contenuto)

### **Validazione**

Sposta le logiche di validazione dai controller alle Request class.

Male:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

Buono:

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[🔝Torna ai contenuti](#contenuto)

### La logica di business dovrebbe stare nelle classi di servizio

Un controller deve avere una sola responsabilità, quindi sposta la logica di business dai controller alle classi di servizio.

Male:

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

Buono:

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[🔝Torna ai contenuti](#contenuto)

### **Non ripeterti (DRY: Don't Repeat Yourself)**

Riutilizzare il codice quando è possibile. Il Principio di Singola Responsabilità (SRP) ti aiuta a evitare la duplicazione. Inoltre, riutilizza i template blade, usa gli eloquenti scopes, ecc.

Male:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Buono:

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[🔝Torna ai contenuti](#contenuto)

### Favorisci l'utilizzo dei Model Eloquent rispetto al Query Builder e alle query SQL raw. Preferisci le Collection agli array

Eloquent ti consente di scrivere codice leggibile e manutenibile. Inoltre, Eloquent ha ottimi strumenti integrati come eliminazioni soft, eventi, scopes, ecc.

Male:

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Buono:

```php
Article::has('user.profile')->verified()->latest()->get();
```

[🔝Torna ai contenuti](#contenuto)

### **Assegnazione di massa**

Male:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Add category to article
$article->category_id = $category->id;
$article->save();
```

Buono:

```php
$category->article()->create($request->validated());
```

[🔝Torna ai contenuti](#contenuto)

### Non eseguire query nei template Blade e utilizzare l'eager loading (problema N + 1)

Male (per 100 utenti, verranno eseguite 101 query DB):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Buono (per 100 utenti, verranno eseguite 2 query DB):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[🔝Torna ai contenuti](#contenuto)

### Commenta il tuo codice, ma cerca anche di rendere autoesplicativi i nomi di metodi e variabili

Male:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Meglio:

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

Buono:

```php
if ($this->hasJoins())
```

[🔝Torna ai contenuti](#contenuto)

### Non inserire JS e CSS nei template Blade e non inserire HTML nelle classi PHP

Male:

```php
let article = `{{ json_encode($article) }}`;
```

Meglio:

```php
<input id="article" type="hidden" value='@json($article)'>

Or

<button class="js-fav-article" data-article='@json($article)'>{{ $article->name }}<button>
```

In un file Javascript:

```javascript
let article = $('#article').val();
```

Il modo migliore è utilizzare il pacchetto PHP-JS specializzato per trasferire i dati.

[🔝Torna ai contenuti](#contenuto)

### Usa file di configurazione e lingua, costanti anziché testo nel codice

Male:

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Buono:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[🔝Torna ai contenuti](#contenuto)

### Utilizza gli strumenti standard Laravel accettati dalla community

Favorisci l'utilizzo delle funzionalità integrate in Laravel e i pacchetti della community anziché utilizzare pacchetti e strumenti di terze parti. Altrimenti, qualsiasi sviluppatore che lavorerà con la tua app in futuro dovrà imparare nuovi strumenti. Inoltre, le possibilità di ottenere aiuto dalla comunità Laravel sono significativamente inferiori quando si utilizza un pacchetto o uno strumento di terze parti. Non far pagare il tuo cliente per quello.

Task | Strumenti standard | Strumenti di terze parti
------------ | ------------- | -------------
Autorizzazione | Policies | Entrust, Sentinel e altri pacchetti
Compiling assets | Laravel Mix | Grunt, Gulp, 3rd party packages
Ambiente di sviluppo | Laravel Sail, Homestead | Docker
Distribuzione | Laravel Forge | Deployer e altre soluzioni
Test unitari | PHPUnit, Mockery | Phpspec
Test del browser | Laravel Dusk | Codeception
DB | Eloquent | SQL, Doctrine
Templates | Blade | Twig
Lavorare con i dati | Laravel Collections | Array
Convalida dei form | Request classes | Pacchetti di terze parti, convalida nel controller
Autenticazione | Incorporato | Pacchetti di terze parti, la tua soluzione
Autenticazione API | Laravel Passport, Laravel Sanctum | Pacchetti JWT e OAuth di terze parti
Creazione dell'API | Incorporato | API Dingo e pacchetti simili
Lavorare con la struttura DB | Migrazioni | Lavorare direttamente con la struttura DB
Localizzazione | Incorporato | Pacchetti di terze parti
Interfacce utente in tempo reale | Laravel Echo, Pusher | Pacchetti di terze parti e funzionamento diretto con WebSocket
Generazione di dati di test | Seeder classes, Model Factories, Faker | Creazione manuale dei dati di test
Pianificazione delle attività | Utilità di pianificazione Laravel | Script e pacchetti di terze parti
DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[🔝Torna ai contenuti](#contenuto)

### **Segui le naming convention di Laravel**

 Seguire [Standard PSR](http://www.php-fig.org/psr/psr-2/).
 
 Inoltre, segui le convenzioni di denominazione accettate dalla comunità Laravel:

Cosa | Come | Buono | Male
------------ | ------------- | ------------- | -------------
Controller | singolare | ArticleController |~~ArticlesController~~
Route | plurale | articles/1 | ~~article/1~~
Named route | snake_case con notazione punto | users.show_active | ~~users.show-active, show-active-users~~
Model | singolare | Utente | ~~Users~~
Relazione hasOne o belongsTo | singolare | articleComment |~~articleComments, article_comment~~
Tutte le altre relazioni | plurale | articleComments | ~~articleComment, article_comments~~
Tabella | plurale | commenti_articolo | ~~article_comment, articleComments~~
Tabella pivot | nomi di modelli singolari in ordine alfabetico | user_user | ~~user_article, articles_users~~
Colonna della tabella | snake_case senza nome modello | meta_title |~~Meta Title; articolo meta_title~~
Proprietà del Model | snake_case | $model->created_at |~~$model->createdAt~~
Foreign key | nome modello singolare con suffisso _id | article_id | ~~ArticleId, id_article, articles_id~~
Chiave primaria | - | id |~~custom_id~~
Migration | - | 2017_01_01_000000_create_articles_table |~~2017_01_01_000000_articles~~
Metodo | camelCase | getAll | ~~get_all~~
Metodo nel resource controller | [tavolo](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Metodo nella test class | camelCase | testGuestCannotSeeArticle |~~test_guest_cannot_see_article~~
Variabile | camelCase | $articoliWithAuthor |~~$articles_with_author~~
Collection | descrittivo, plurale | $activeUsers = Utente::active()->get() | ~~$active, $data~~
Oggetto | descrittivo, singolare | $activeUser = User::active()->first() | ~~$users, $obj~~
Indice file di configurazione e lingua | snake_case | articles_enabled |~~ArticlesEnabled; articles-enabled~~
View | kebab-case | show-filtered.blade.php | ~~showFiltered.blade.php, show_filtered.blade.php~~
Config | snake_case | google_calendar.php |~~googleCalendar.php, google-calendar.php~~
Contratto (interfaccia) | aggettivo o sostantivo | AuthenticationInterface | ~~Authenticatable, IAuthentication~~
Trait | aggettivo | Notificabile | ~~NotificationTrait~~

[🔝Torna ai contenuti](#contenuto)

### Utilizza la sintassi più breve e più leggibile ove possibile

Male:

```php
$request->session()->get('cart');
$request->input('name');
```

Buono:

```php
session('cart');
$request->name;
```

Più esempi:

Common syntax | Shorter and more readable syntax
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? null : $object->relation->id` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[🔝Torna ai contenuti](#contenuto)

### **Utilizzare il container IoC o i Facades invece di istanziare nuove classi**

La sintassi new Class crea un accoppiamento stretto tra le classi e complica i test. Utilizzare invece il container IoC o i Facades.

Male:

```php
$user = new User;
$user->create($request->validated());
```

Buono:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->validated());
```

[🔝Torna ai contenuti](#contenuto)

### Non prelevare direttamente i dati dal file `.env`

Passa i dati presenti nell'.env file ai file di configurazione e quindi usa l'helper `config ()` per prelevare i dati all'interno dell'applicazione.

Male:

```php
$apiKey = env('API_KEY');
```

Buono:

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

[🔝Torna ai contenuti](#contenuto)

### **Memorizza le date nel formato standard. Utilizza gli accessors e i mutators per modificare il formato della data**

Male:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

Buono:

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at'];
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

[🔝Torna ai contenuti](#contenuto)

### **Altre buone pratiche**

Non inserire mai alcuna logica nei file di route.

Ridurre al minimo l'utilizzo di vanilla PHP nei template Blade.

[🔝Torna ai contenuti](#contenuto)
