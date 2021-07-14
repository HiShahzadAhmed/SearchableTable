# Searchable Table

Searchable Table for Server side processing

## Installation

You can install the package via composer:

```bash
composer require spatie/laravel-searchable
```



## Usage

### Preparing your models

You'll need to add a `getSearchResult` method to each searchable model that must return an instance of `SearchResult`. Here's how it could look like for a sample Item model.

```php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Spatie\Searchable\Searchable;
use Spatie\Searchable\SearchResult;

class Item extends Model implements Searchable
{
    use HasFactory;

    public function getSearchResult(): SearchResult
     {
         return new \Spatie\Searchable\SearchResult(
            $this,
            $this->name,
         );
     }


}

```
NOTE: Change $this->YOUR_OWN_MODEL_COLUMN to your model column that have more priority e.g ($this->name, $this->title) etc.
```php
return new \Spatie\Searchable\SearchResult(
       $this,
       $this->YOUR_OWN_MODEL_COLUMN,
    );
```


### Preparing your Routes & Controller for searching

You'll need to create a method to link from web.php file for searching. Make a route e.g.

```bash
Route::get('items','ItemsController@items')->name('items');
```

Your Controller will be something like this with method

```php

<?php
  
namespace App\Http\Controllers;
  
use Illuminate\Http\Request;
use App\Models\Item;
use Spatie\Searchable\Search;


class ItemsController extends Controller
{


    public function items(Request $request)
    {

        #Request from AJAX and RENDER Result View
        if ($request->ajax()) 
        {

            #GO FOR SPECIFIC MODEL FOR SEARCHABLE INTERFACES to add getSearchResult() METHOD
            $searchResults = (new Search())
               ->registerModel(Item::class, ['name', 'address'])
               ->search($request->table_filter_search ?? ' ');

            $searchResults = $searchResults->paginate($request->table_length_limit);
            
            return view('items_result', get_defined_vars());
        }
  
        #Page LOAD Call from main ROUTE and VIEW Return
        return view('items');
    }
}

```
NOTE: Change the Model name in place of ['YOUR MODEL NAME'] & replace fileds name in place of foo, bar of coressponding array like ['title', 'address']
```php
->registerModel(['YOUR MODEL NAME']::class, ['foo', 'bar'])
```




### Preparing for Render and Loading Views

Here we will need 2 blade files e.g
1. items.blade.php (For First time loading)
2. items_result.blade.php (For rendering data via AJAX call)


### items.blade.php
The file will be something like this:

```blade

    <style>
        .pagination 
        {
          display: inline-flex;
          margin: 0 auto;
        }
        .overlay-bg
        {
            position: absolute;
            z-index: 99;
            background-color: rgba(0, 0, 0, 0.1); 
            height:100%;
            width: 100%;
            padding: 0;
        }
    </style>
    
    <div class="overlay-bg d-none"></div>
      <div class="container py-4">

          <div class="card shadow-sm">
              <div class="card-body">
                  <div class="row">
                      <div class="col-lg-6 mb-2">
                          <span>Show </span>
                          <select name="table_length_limit" class="table_length_limit">
                              <option value="15">15</option>
                              <option value="25">25</option>
                              <option value="50">50</option>
                              <option value="100">100</option>
                          </select>
                           <span>entries</span>
                      </div>
                      <div class="col-lg-6 text-right">
                           <span>Search: </span>
                           <input type="text" name="table_filter_search" class="table_filter_search" value="{{ $request->table_filter_search ?? '' }}" class="">
                      </div>
                  </div>
                  <!--RECORD WILL BE APPEND HERE in #append-record FROM RENDERED VIEW VIA AJAX-->
                  <div id="append-record"></div>
              </div>
          </div>

      </div>
      
      <script>
        // SET DEFAULT TABLE LENGTH LIMIT
        $table_length_limit  = 15;



        // LOAD RECORD ON PAGE LOAD FIRST TIME
        $(document).ready(function(){
            loadRecords();
        });


        // LOAD RECORD FUNCTION
        function loadRecords(){
            $.ajax({
                    url: '?table_length_limit=' + $table_length_limit,
                    type: "get",
                    datatype: "html"
                }).done(function(data){
                    $("#append-record").empty().html(data);
                }).fail(function(jqXHR, ajaxOptions, thrownError){
                    loadRecords();
                });
        }


        // GET DATA FROM SERVER EVENT
        function getData(page){
            $.ajax({
                    url: '?page=' + page + '&table_length_limit=' + $('.table_length_limit').val(),
                    type: "get",
                    datatype: "html"
                }).done(function(data){
                    $("#append-record").empty().html(data);
                }).fail(function(jqXHR, ajaxOptions, thrownError){
                    alert('No response from server');
                });
        }


        // PAGINATION CLICK EVENT
        $(document).on('click', '.pagination a',function(event){

            event.preventDefault();

            $('li').removeClass('active');
            $(this).parent('li').addClass('active');

            var myurl = $(this).attr('href');
            var page=$(this).attr('href').split('page=')[1];
            getData(page);

        });



        // TABLE LENGTH CHANGE EVENT
        $(document).on('change', '.table_length_limit',function(event){
            $('.overlay-bg').removeClass('d-none');
            $.ajax(
            {
                url: '?table_length_limit=' + $(this).val(),
                type: "get",
                datatype: "html"
            }).done(function(data){
                $("#append-record").empty().html(data);
                $('.overlay-bg').addClass('d-none');
            }).fail(function(jqXHR, ajaxOptions, thrownError){
                  alert('No response from server');
            });

        });    

        //TABLE FILTER SEARCH BOX EVENT
        $(document).on('keyup', '.table_filter_search',function(event){

            $('.overlay-bg').removeClass('d-none');

            $.ajax({
                    url: '?table_filter_search='+$(this).val()+'&table_length_limit=' + $('.table_length_limit').val(),
                    type: "get",
                    datatype: "html"
            }).done(function(data){
                    $("#append-record").empty().html(data);
                    $('.overlay-bg').addClass('d-none');
            }).fail(function(jqXHR, ajaxOptions, thrownError){
                    alert('No response from server');
            });

        });


    </script>

```


### items_result.blade.php
The file will be something like this:

```blade

        <table class="table table-bordered table-striped table-sm">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Address</th>
                    <th class="text-right">Action</th>
                </tr>
            </thead>
            <tbody id="tag_container">

                @forelse($searchResults as $value)
                <tr>
                    <td>
                        @if($value->searchable->id & 1)
                            <span class="badge badge-dark">{{ $value->searchable->id }}</span>
                        @else
                            <span class="badge badge-info">{{ $value->searchable->id }}</span>
                        @endif
                    </td>
                    <td>{{ $value->searchable->name }}</td>
                    <td>{{ $value->searchable->address ?? 'N/A' }}</td>
                    <td class="text-right">
                        <div class="btn-group">
                            <a href="#" class="btn btn-primary">Edit</a>
                            <a href="#" class="btn btn-danger">Remove</a>
                        </div>
                    </td>
                </tr>
                @empty
                    <tr class="text-center" >
                        <td colspan="4">No records were found right now!</td>
                    </tr>
                @endforelse

            </tbody>
        </table>

        <div class="row">
            <div class="col-lg-6 text-left">
                <p>Showing {{ $searchResults->firstItem() }} to {{ $searchResults->lastItem() }} of {{ $searchResults->total() }} entries</p>
            </div>
            <div class="col-lg-6 text-right">
                {{ $searchResults->onEachSide(1)->links() }}
            </div>
        </div>


```

## Note
Don't forget to change pagination of Laravel from Tailwind to Bootstrap. To change this go to your 
App/Http/Providers/AppServiceProvider.php

Include the namespace

```bash
use Illuminate\Pagination\Paginator;
use Illuminate\Pagination\LengthAwarePaginator;
```
After that copy the below line and add to boot method of AppServiceProvider

```bash
Paginator::useBootstrap();
```



