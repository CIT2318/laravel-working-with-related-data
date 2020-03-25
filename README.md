# Laravel - Working with related data

This practical follows on from the previous Laravel practical. We will create an additional database table and look at techniques for working with related data.

## Creating the migration
### Migration for the *certificates* table
* Using artisan, create a migration for a *certificates* table
```
php artisan make:migration create_certificates_table --create=certificates
```
* Open the migration in a text-editor. We will keep things simple and only define *name* and *description* columns.

```
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateCertificatesTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('certificates', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name',2);
            $table->string('description');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('certificates');
    }
}
```

### Defining a foreign key for the *films* table
We could edit the existing *films* table migration, but instead we will simply create a new migration that will define an additional column for the *films* table.

* Run the following Artisan command
```
php artisan make:migration add_certificate_id_to_films_table --table=films
```

* Edit the resulting file so it looks like the following:

```
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class AddCertificateIdToFilmsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('films', function (Blueprint $table) {
            $table->integer('certificate_id')->unsigned();
            $table->foreign('certificate_id')->references('id')->on('certificates');
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('films', function (Blueprint $table) {
            $table->dropForeign('films_certificate_id_foreign');
            $table->dropColumn('certificate_id');
        });
    }
}
```

* This migration simply defines an additional column and specifies a foreign key for the existing films table. Again, see the Laravel website (https://laravel.com/docs/migrations) for complete explanations.
* If we run these migration we will get an error because there is already a films table in our database. We need to remove the existing tables associated with the application and then fire off the new migrations. First, remove the existing tables by running the following Artisan command:

```
php artisan migrate:reset
```

* Now we are ready to run our migrations
```
php artisan migrate
```
* Check phpMyAdmin, the tables should be set up along with a foreign key in the films tables.

## Seeding
This is fairly straightforward. Using Artisan enter the following command:
```
php artisan make:seeder CertificatesTableSeeder
```
Open the newly generated file and add some seeds i.e.
```
use Illuminate\Database\Seeder;

class CertificatesTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
         DB::table('certificates')->insert(['name' => 'U','description' => 'Suitable for all.']);
         DB::table('certificates')->insert(['name' => 'PG','description' => 'Some scenes may be unsuitable for young children.']);
         DB::table('certificates')->insert(['name' => '12','description' => 'Suitable for people aged 12 and over.']);
         DB::table('certificates')->insert(['name' => '15','description' => 'Suitable for people aged 15 and over.']);
         DB::table('certificates')->insert(['name' => '18','description' => 'Suitable for people aged 18 and over.']);

    }
}
```

* Next modify the existing films seeder so that we specify the certificate_id for each film. I've added an extra film so that they don't all reference certificate 4.

```
use Illuminate\Database\Seeder;

class FilmsTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        DB::table('films')->insert(['title' => 'Jaws','year' => '1975', 'duration'=>124, 'certificate_id'=>4]);
        DB::table('films')->insert(['title' => 'Winter\'s Bone','year' => '2010', 'duration'=>100, 'certificate_id'=>4]);
        DB::table('films')->insert(['title' => 'Do The Right Thing','year' => '1989', 'duration'=>120, 'certificate_id'=>4]);
        DB::table('films')->insert(['title' => 'The Incredibles','year' => '2004', 'duration'=>115, 'certificate_id'=>1]);
    }
}

```

* Finally edit *DatabaseSeeder.php* to run both seeders.

```
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        // $this->call(UsersTableSeeder::class);
        $this->call(CertificatesTableSeeder::class);
        $this->call(FilmsTableSeeder::class);
    }
}
```

* To seed the database, using Artisan enter:
```
php artisan db:seed
```
* Open phpmyadmin and check the seeding has worked.

##Creating the Model class
Enter the following Artisan commands to create the Certificate model class:

```
php artisan make:model Certificate
```

## Defining the relationship between Certificate and Film
In this scenario there is a one-to-many relationship between Certificate and Film. We can specify this relationship using Eloquent, and Laravel will automatically assign an array of Film objects to a Certificate instance.

* Modify the Certificate class to add a films method:
```
namespace App;

use Illuminate\Database\Eloquent\Model;

class Certificate extends Model
{
    public function films()
    {
        return $this->hasMany('App\Film');
    }
}
```
* We also need to define the inverse of this relationship so that Eloquent can assign the relevant Certificate to a Film instance. Add a certificate method to the Film class.

```
namespace App;

use Illuminate\Database\Eloquent\Model;

class Film extends Model
{
    public function certificate()
    {
        return $this->belongsTo('App\Certificate');
    }
}
```
* *hasMany* and *belongsTo* are part of Eloquent, the Film and Certificate classes extend the base Model class so have access to these methods. You can read more at https://laravel.com/docs/eloquent-relationships#one-to-many.
* That's all we have to do to set up the ORM for Film and Certificate. To check this is working, make some changes to the details view so that you also output the film's certificate. Here's an example:
```
@extends('layouts.master')
@section('title', 'Film details')
@section('content')
<h1>{{$film->title}}</h1>
<ul>
<li>Year: {{$film->year}}</li>
<li>Duration: {{$film->duration}}mins</li>
<li>Certificate:{{$film->certificate->name}}</li>
</ul>
@endsection
```

## Updating the rest of the site
The list of films and delete functions should still work fine. You will need to make some changes to the 'create' functionality so the user can specify a certificate for the film they add. Open FilmController.php and modify the *create* method so it looks like the following:

```
function create()
{
   $certificates = Certificate::all();
    return view('film/create-view',['certificates'=>$certificates]);
}
```

You will also need to import the Certificate class with a use statement at the top of the file:

```
use App\Certificate;
```

* It should be clear what is happening here. We need a list of certificates from the database so we can populate the form.
* Next, Modify the create view, add the following just before the submit button. Again, it should be clear what this does. We generate a radio button for each certificate.

```
<fieldset>
<legend>Select the film's certificate</legend>
@foreach($certificates as $certificate)
    <div>
        <label for="dirBtn{{$certificate->id}}">
        <input id="dirBtn{{$certificate->id}}" type="radio" name="certificate" value="{{$certificate->id}}">
        {{$certificate->name}}
        </label>
    </div>
@endforeach
</fieldset>
```

* Test this works. The user should be able to see options for a choice of certificate on the add film page (but you'll get an error if you try and add a film).

### Inserting into the database.
We follow exactly the same stages as when working with plain PHP but we will use Eloquent to save some typing. We need to:
* Create a Film object
* Associate this Film object with a certificate.
* Persist the film in the database.

Modify the save method so it looks like the following:
```
function save(Request $request)
{
    $this->validate($request, [
        'title' => 'required|max:100',
        'year' => 'required|integer|min:1900',
        'duration' => 'required|integer|min:1|max:300',
    ]);
    //create the new film
    $film = new Film();
    $film->title = $request->title;
    $film->year=$request->year;
    $film->duration=$request->duration;
    //assign certificate to film
    $certificate = Certificate::find($request->certificate);
    $film->certificate()->associate($certificate);

    $film->save();
    return redirect('list');
}
```
* Note, that we also have added a validation rule for the certificate.
* Test this works, you should now be able to add films and the database should update.

## On your own
* The above only deals with one-to-many relationships. Have a go at creating many-to-many relationships e.g. by associating films with genres.
  * You will need to go through the same steps as above - migrations, seeding etc.  The laravel documentation has info on many-to-many relationships - https://laravel.com/docs/eloquent-relationships#many-to-many.
