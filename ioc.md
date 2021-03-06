git f53f291f67507108dd0f5772a7eb33bc6458d840

---

# Inversion of Control (обратное управление, IoC)

- [Введение](#introduction)
- [Основы использования](#basic-usage)
- [Где писать код ?](#where-to-register)
- [Автоматическое определение](#automatic-resolution)
- [Практическое использование](#practical-usage)
- [Сервис-провайдеры](#service-providers)
- [События контейнераs](#container-events)

<a name="introduction"></a>
## Введение

Класс-контейнер обратного управления IoC (Inversion of Control) Laravel - мощное средство для управления зависимостями классов. Внедрение зависимостей - это способ исключения вшитых (hardcoded) взаимосвязей классов. Вместо этого зависимости определяются во время выполнения, что даёт бОльшую гибкость благодаря тому, что они могут быть легко изменены.

Понимание IoC-контейнера Laravel необходимо для построения больших приложений, а также для внесения изменений в код ядра самого фреймворка.

<a name="basic-usage"></a>
## Основы использования

Есть два способа, которыми IoC-контейнер разрешает зависимости: через функцию-замыкание и через автоматическое определение. 

Для начала давайте посмотрим замыкания. Первым делом нечто ("тип") должно быть помещёно в контейнер.

#### Помещение в контейнер

	App::bind('foo', function($app)
	{
		return new FooBar;
	});

#### Извлечение из контейнера

	$value = App::make('foo');

При вызове метода `App::make` вызывается соответствующее замыкание и возвращается результат её вызова.

Иногда вам может понадобиться поместить в контейнер тип, который должен быть создан только один раз, и чтобы все последующие вызовы возвращали бы тот же объект.

#### Помещение shared ("разделяемого") типа в контейнер

	App::singleton('foo', function()
	{
		return new FooBar;
	});

Вы также можете поместить уже созданный экземпляр объекта в контейнер, используя метод `instance`:

#### Помещение готового экземпляра в контейнер

	$foo = new Foo
	App::instance('foo', $foo);

<a name="where-to-register"></a>
## Где писать код ?

Вышеописанные биндинги (связывание) в IoC, как и регистраторы обработчиков событий или фильтры роутов, можно писать в старт-файле `app/start/global.php`, или, например, в файле `app/ioc.php` (имя непринципиально) и подключать его (include) в `app/start/global.php`. Если же вам ближе ООП-подход, или у вас в приложении много IoC-биндингов и вы хотите разложить их по категориям, а не держать все в одном файле, вы можете описать ваши биндинги в ваших [сервис-провайдерах](#service-providers).

<a name="automatic-resolution"></a>
## Автоопределение класса

Контейнер IoC достаточно мощен, чтобы во многих случаях определять классы автоматически, без дополнительной настройки.

#### Автоопределение класса

	class FooBar {

		public function __construct(Baz $baz)
		{
			$this->baz = $baz;
		}

	}

	$fooBar = App::make('FooBar');

Обратите внимание, что даже без явного помещения класса FooBar в контейнер он всё равно был определён и зависимость `Baz` была автоматически внедрена в него.

Если тип не был найден в контейнере, IoC будет использовать возможности PHP для изучения класса и [контроля типов](http://www.php.net/manual/ru/language.oop5.typehinting.php) в его конструкторе. С помощью этой информации контейнер сам создаёт экземпляр класса.

Однако в некоторых случаях класс может принимать экземпляр интерфейса, а не сам объект. В этом случае нужно использовать метод `App::bind` для извещения контейнра о том, какая именно зависимость должна быть внедрена.

#### Связывание интерфейса и реализации

	App::bind('UserRepositoryInterface', 'DbUserRepository');

Теперь посмотрим на следующий контроллер:

	class UserController extends BaseController {

		public function __construct(UserRepositoryInterface $users)
		{
			$this->users = $users;
		}

	}

Благодаря тому, что мы связали `UserRepositoryInterface` с "настоящим" классом, `DbUserRepository`, он будет автоматически встроен в контроллер при его создании.

<a name="practical-usage"></a>
## Практическое использование

Laravel предоставляет несколько возможностей для использования контейнера IoC для повышения гибкости и тестируемости вашего приложения. Основной пример - внедрение зависимости при использовании контроллеров. Все контроллеры извлекаются из IoC, что позволяет вам использовать зависимости на основе контроля типов в их конструкторах - ведь они будут определены автоматически.

#### Контроль типов для указания зависимостей контроллера

	class OrderController extends BaseController {

		public function __construct(OrderRepository $orders)
		{
			$this->orders = $orders;
		}

		public function getIndex()
		{
			$all = $this->orders->all();

			return View::make('orders', compact('all'));
		}

	}

В этом примере класс `OrderRepository` автоматически встроится в контроллер. Это значит, что при использовании [юнит-тестов](/docs/{{version}}/testing) класс-заглушка ("mock") для `OrderRepository` может быть добавлен в контейнер, таким образом легко имитируя взаимодействие с БД.

[Фильтры](/docs/{{version}}/routing#route-filters), [вью-композеры](/docs/{{version}}/responses#view-composers) и [обработчики событий](/docs/{{version}}/events#using-classes-as-listeners) могут также извлекаться из IoC. При регистрации этих объектов просто передайте имя класса, который должен быть использован:

#### Другие примеры использования IoC

	Route::filter('foo', 'FooFilter');

	View::composer('foo', 'FooComposer');

	Event::listen('foo', 'FooHandler');

<a name="service-providers"></a>
## Сервис-провайдеры

Сервис-провайдеры (service providers, "поставщики услуг") - отличный способ группировки схожих регистраций в IoC в одном месте. Их можно рассматривать как начальный запуск компонентов вашего приложения. Внутри поставщика услуг вы можете зарегистрировать драйвер авторизации, классы-хранилища вашего приложения или даже собственную консольную Artisan-команду.

БОльшая часть компонентов Laravel включает в себя свои сервис-провайдеры. Все зарегистрированные сервис-провайдеры в вашем приложении указаны в массиве `providers` файла настроек `app/config/app.php`.

Для создания нового сервис-провайдера просто наследуйте класс `Illuminate\Support\ServiceProvider` и определите метод `register`.

#### Создание сервис-провайдера

	use Illuminate\Support\ServiceProvider;

	class FooServiceProvider extends ServiceProvider {

		public function register()
		{
			$this->app->bind('foo', function()
			{
				return new Foo;
			});
		}

	}

Заметьте, что внутри метода `register` IoC-контейнер приложения доступен в свойстве `$this->app`. Как только вы создали провайдера и готовы зарегистрировать его в своём приложении, просто добавьте его в массивe `providers` файла настроек `app/config/app.php`.

Кроме этого, вы можете зарегистрировать его "на лету", используя метод `App::register`:

#### Регистрация поставщика услуг во время выполнения

	App::register('FooServiceProvider');

<a name="container-events"></a>
## События контейнера

IoC-контейнер запускает событие каждый раз при извлечении объекта. Вы можете отслеживать его с помощью метода `resolving`:

#### Регистрация обработчика события

	App::resolvingAny(function($object)
	{
		//
	});

	App::resolving('foo', function($foo)
	{
		//
	});

Созданный объект передаётся в функцию-замыкание в виде параметра.