# Laravel-Controller sichern mit Form-Request und Gate
In diesem Beiträge erkläre ich, wie ich mit Hilfe von [Form-Requests](https://laravel.com/docs/master/validation#form-request-validation) und [Gates](https://laravel.com/docs/master/authorization#gates) die Authorizierung meiner Laravel-Controller vereinfacht habe.  
Das macht die Berechtigungen im System übersichtlicher und ist leicht zu testen. 

## TL;DR 
[Beispiel Repository](https://github.com/KayDomrose/larave-request-gate-example)

## Früher war alles besser ...
... außer mein Code.  
Da habe ich nämlich die Authorizierungs-Logik direkt in meine Controller gepackt und dort die Exceptions geworfen.  
Das hat auch erstmal ganz gut geklappt, hat aber zwei entscheidene Nachteile:

1. Schwer zu testen  
Laravel-Controller sind verhältnismäßig komplexe Gebilde und übernehmen viele Aufgaben auf einmal (Request, Weiterverarbeitung, Response).  
Dadurch werden Unit-Tests dafür schnell umfangreich und müssen mit vielen Parametern und Zuständen arbeiten.  
Gerade bei der Authorizierung sollten die Tests aber lückenlos und umfangreich sein, um alle Möglichkeiten abzudecken. 

2. Unübersichtlich  
Die Logik für die Authorizierung ist in allen Controllern verstreut. So ist es schwer einen schnellen Überblick zu bekommen, welche Berechtigungen existieren und wie diese ermittelt werden.

## Die Rettung durch Form-Requests und Gates
Nach einigem Herumprobieren habe ich eine gute und saubere Lösung gefunden, die für mich sehr gut funktioniert.

Die Authorizierung wird vom Controller über einen eigenen [Form-Request](https://laravel.com/docs/master/validation#form-request-validation) an ein [Gate](https://laravel.com/docs/master/authorization#gates) weitergeleitet.

Für jeden Schritt lassen sich recht einfach Unit-Tests umsetzen und alle Berechtigungen werden an einer zentralen Stelle konfiguriert.

## Beispiel
Ich will alle Informationen zu einem Nutzer bekommen. Die URL dafür sähe zum Beispiel so aus: `/user/{user}`.  

### Controller
```php
// /app/Http/Controllers/UserController.php

// ...

class UserController extends Controller 
{
    public function read(UserReadRequest $request, User $user)
    {
        return $user;
    }
}
```

Als ersten Parameter nehme ich meinen individuellen Form-Request für diese Methode.  
Dieser wird von Laravel ohne weiters zutun instanziert ([Dependency Injection](https://laravel.com/docs/master/controllers#dependency-injection-and-controllers)) und die darin enthalte Authorizierung und Validierung geprüft.  
Die Methode `read` wird also erst ausgeführt, wenn damit keine Probleme vorliegen.

Durch [Route-Model-Binding](https://laravel.com/docs/master/routing#route-model-binding) wird außerdem direkt der richtige `User` geladen oder ein 404-Fehler geworfen, wenn der Nutzer nicht gefunden wurde.

In `read` kann ich mich nun ganz auf meine eigentiche Logik konzentrieren und brauche keine zusätzlichen Abfragen.  

#### Controller-Test
Um zu prüfen ob der Controller auch wirklich den richtigen Request ausführt, kann ein einfacher Unit-Test helfen.  
Da bei fehlerhafter Authorizierung potenziell sensible Informationen an Unbefugte geraten könne, ist das besonders wichtig.

```php
// /tests/Unit/Controllers/UserControllerTest.php

// ...

class UserControllerTest extends TestCase
{
    use HttpTestAssertions;

    public function test_controller_calls_correct_request()
    {
        $this->assertActionUsesFormRequest(
            UserController::class,
            'read',
            UserReadRequest::class
        );
    }
}

```

Für die Prüfung ob eine bestimmte Methode in einem Controller einen bestimmten Request benutzt, benutze ich [HttpTestAssertions](https://github.com/jasonmccreary/laravel-test-assertions).  
Dort gebe ich den zu prüfenden Controller und die Methode an, sowie den Request, der aufgerufen werden soll.  

Wenn dieser Test besteht kann ich sicher sein, dass immer mein individueller Request verwendet wird.

### Request
Ich benennen meine Requests im Format `<ControllerName><MethodenName>Request`. So kann ich einerseits schnell danach suchen und erkenne direkt wo der Request eingesetzt wird.  

```bash
php artisan make:request User/UserReadRequest
```

```php
// /app/Http/Requests/User/UserReadRequest.php

// ...

class UserReadRequest extends FormRequest
{
    public function authorize()
    {
        return Gate::forUser($this->user())->allows('user.read', $this->user);
    }
    
    // ...
}
```

Hier passiert nichts weiter außer das die Authorizierung an das Gate weitergeleitet wird.  
Mit `Gate::forUser` stelle ich sicher, dass das Gate mit dem aktuellen Nutzer, der den Request angestoßen hat, ausgeführt wird.  
Alternativ zu `$this->user()` könnte ich auch `Auth::user()` verwenden, aber ich finde es sauberer im Kontext des aktuellen Requests zu bleiben. Der Rückgabewert von `$this->user()` ist `User` oder `null`.  
Wenn `forUser` nicht explizit verwendet wird, benutzt Laravel den aktuell authentifizierten Nutzer. Das geht meistens auch, spätestens bei den Unit-Tests wirds dann aber umständlicher. Ich gebe es daher lieber explizit mit an, so bin ich auf der sicheren Seite.

Die Methode `allows` verlangt als ersten Parameter einen `string`, der die auszuführende Aktion beschreibt. Ich nehme meistens sowas wie `<model>.<aktion>`.  
Wenn keine Gate-Definition für die Aktion gefunden wurde, ist der Rückgabewert immer `false`. Das kann leicht bei Schreibfehlern passieren.  
Als zweiten Parameter kamn zusätzlich ein Objekt übergeben werden, wenn die Berechtigung davon abhängig ist. In diesem Fall ist die Authorizierung abhängig davon, welcher Nutzer gelesen werden soll, daher übergebe ich ihn als zweites.  
Da ich das Model per Route-Model-Binding aufrufe, kann ich mit `$this->user` auch im Request darauf zugreifen (`'/{model}'` => `$this->model`).

Wichtiger Unterschied:  
`$this->user()` = Aktuell angemeldeter User  
`$this->user` = User dessen Informationen abgerufen werden soll

Der Rückgabewert von `authorize()` wird als Boolean behandelt. Danach wird entschieden, ob der Request authoriziert ist und der Controller ausgeführt wird oder ein 403-Fehler (`AuthenticationException`) geworfen wird.  
Die Authorizierungslogik könnte also auch direkt hier stattfinden. Die Gates können aber auch noch anderen Stellen, zum Bespiel im Blade-Template, wiederverwendet werden.  

#### Request-Test
Ebenso wie im Controller muss geprüft werden, dass die Authorizierung an die zuständige Stelle (das Gate) mit der richtigen Aktion, dem aktuellen Nutzer und dem Nutzer, dessen Informationen gefordert werden, aufgerufen wird.

```bash
php artisan make:test --unit Requests/User/UserReadRequestTest
```

```php
// /test/Unit/Requests/User/UserReadRequestTest.php

// ...
//use PHPUnit\Framework\TestCase;
use Tests\TestCase;

class UserReadRequestTest extends TestCase
{
    public function test_authorize_calls_gate()
    {
        $request = new UserReadRequest();
        $request->setUserResolver(function () {
            return factory(User::class)->make([
                'name' => 'currentUserName'
            ]);
        });
        $request->user = factory(User::class)->make([
            'name' => 'targetUserName',
        ]);

        $gate = Mockery::mock(GateContract::class);
        $gate
            ->shouldReceive('allows')
            ->with(
                'user.read',
                Mockery::on(function ($user) {
                    return $user->name === 'targetUserName';
                })
            )
            ->once();

        Gate::shouldReceive('forUser')
            ->with(Mockery::on(function ($user) {
                return $user->name === 'currentUserName';
            }))
            ->andReturn($gate)
            ->once();

        $request->authorize();
    }
}
```
Man sieht schon das der Test für eine so simple Logik recht umfangreich wird, alleine schon weil der Request ein verhältnismäßig umständliches Setup hat.  
Auch deswegen ist es sinnvoll im Request nur die Weiterleitung an das Gate zu machen statt die komplette Logik. 

Im ersten Schritt wird der Request erzeugt. Dabei benutze ich [Factories](https://laravel.com/docs/master/database-testing#writing-factories) um zwei Test-Nutzer zu erstellen und dem Request anzuhängen. Ich gebe denen jeweils einen eindeutigen Namen um im Test später bestimmen zu können, dass die richtigen Nutzer an den richtigen Stellen übergeben werden.  
Wichtig: Wenn die Tests via `artisan` erzeugt werden, wird neuerdings von `PHPUnit\Framework\TestCase` erweitert. Da ich hier aber mit Factories arbeite, muss ich stattdessen `Tests\TestCase` verwenden ([#30879](https://github.com/laravel/framework/issues/30879#issuecomment-567456608)).

Dann erzeuge ich ein Mock für `GateContract`, um die `allows`-Methode zu prüfen.  
Der erste Parameter soll `'user.read'` sein, der zweite ein Objekt dessen Eigenschaft `name` dem meines Testnutzers entspricht.So gehe ich sicher, dass hier der Nutzer übergeben wird, dessen Informationen gefragt werden und nicht derjenige, der den Request aufruft.

Da ich die `Facade` für `Gate` verwende, kann ich direkt Erwartungen für `forUser` definieren (auch hier prüfe ich `name` statt einfach ob es sich um ein `User`-Objekt handelt um Verwechslungen auszuschließen).

Nachdem alle Vorbereitungen für den Test abgeschlossen sind, wird `$request->authorize()` ausgeführt. Die Mocks für `forUser` und `allows` würden nun Alarm schlagen, wenn die Methoden nicht oder mit anderen Parametern aufgerufen würden.  
Wenn der Test erfolgreich durchläuft kann ich sicher sein, dass die Authorizierungsfrage korrekt weitergeleitet wurde.

### Gate
Bist du noch bei mir? Ausgezeichnet!  
Du hast es fast geschafft, versprochen.

Nun kann ich mich um darum kümmern die tatsächliche Logik das Gate festzulegen.

Ganz schön viel Zeug für so eine triviale Aufgabe, hm?

Stimmt wohl. Allerdings habe ich so auch in komplexen Anwendungen ein einheitliches und leicht testbares Verfahren um die Authorizierung festzustellen.  

Da ich für gewöhnlich in großen Projekten mit komplexen Berechtigungen arbeite, habe ich für jeden Bereich eine eigenen Gates-Datei, in der ich die Logik zusammenfasse.  
Diese müssen zum Start von Laravel geladen werden.

```php
// /app/Providers/AuthServiceProvider.php

public function boot()
{
    // ...

    UserGate::register();

}
```

Nun zum eigentlichen Gate, in dem die Logik für `User` hinterlegt wird.

```php
// /app/Gates/UserGate.php

class UserGate
{
    public static function register()
    {
        Gate::define('user.read', function(User $currentUser, User $targetUser):bool {
            return $currentUser->getAttribute('id') === $targetUser->getAttribute('id');
        });
    }
}
```

Die Implementierung folgt genau der [Dokumentation zum Schreiben von Gates](https://laravel.com/docs/master/authorization#writing-gates).

Ich definiere die Aktion `'user.read'`. Aus dem Gate-Aufruf im Request erhalte ich als ersten Parameter den Nutzer, der per `forUser` übergeben wurde und als zweiten das Model via `allows('user.read', $user)`.
In diesem Fall vergleiche ich einfach ob die `id` übereinstimmt. Wichtig ist hier nur, dass ein boolscher Wert zurückgegeben wird.

#### Gate-Test
Zu guter Letzt können wir nun endlich die Tests schreiben, um die eigentlichen Berechtigungen zu prüfen.

```bash
php artisan make:test --unit Gates/UserGateTest
```

```php
// /app/test/Unit/Gates/UserGateTest.php

//use PHPUnit\Framework\TestCase;
use Tests\TestCase;

class UserGateTest extends TestCase
{
    use RefreshDatabase;

    public function test_allow_reading_user_when_same_id()
    {
        $currentUser = factory(User::class)->create();

        $result = Gate::forUser($currentUser)->allows('user.read', $currentUser);

        $this->assertTrue($result);
    }

    public function test_dont_allow_reading_user_when_not_same_id()
    {
        $currentUser = factory(User::class)->create();
        $targetUser = factory(User::class)->create();

        $result = Gate::forUser($currentUser)->allows('user.read', $targetUser);

        $this->assertFalse($result);
    }
}
```

Man kann schon sehen, dass das Setup für diese Tests deutlich schlanker ist als in den vorherigen mit den ganzen Mocks.  
Das macht es natürlich viel einfacher die unterschiedlichen Berechtigungsmöglichkeiten zu testen, die in einem realen System vermutlich deutlich umfangreicher sind.

Im vorherigen Request-Test habe ich in der Fabrik temporäre Models erzeugt (`->make()`), die nicht in der Datenbank gespeichert wurden. Dadurch kaufen die Tests scheller.  
Da ich hier aber die IDs vergleichen möchte, muss ich stattdessen `->create()` verwenden. Damit werden die Models tatsächlich in die Datenbank geschrieben und eine ID vergeben.  
Um sicherzugehen, dass die Datenbank auch existiert und die Tabellen angelegt sind, muss ich mit `use RefreshDatabase` arbeiten.

Außerdem muss ebenso wie im vorherigen Test `Tests\TestCase` verwendet werden.

## Zusammenfassung
Damit ist die Authorizierung abgeschlossen.

Durch die Weiterleitung der Authorizierungslogik vom Controller über Request zum Gate habe ich eine zentrale Stelle, an der meine komplette Logik für das ganze System liegt.  
Da ich jeden einzelnen Schritt und zuletzt die Logik im Gate mit Unit-Tests abgedeckt habe kann ich sicher sein, dass meine Controller richtig gesichert sind. Gleichzeit kann ich die selbe Logik auch noch an anderer Stelle, zum Beispiel im Template verwenden.

Ich hoffe du konntest das Eine oder Andere für deine eigenen Projekte mitnehmen.

Das ist mein erster Beitrag dieser Art, daher freue ich mich besonders über konstruktive Kritik! 
