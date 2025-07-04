Step 1: Set up your Models
Assuming you have two tables: invoices and invoice_details.

Create Invoice model if you haven't already:

bash
Copy
Edit
php artisan make:model Invoice
In the Invoice model, you might have something like this:

php
Copy
Edit
// app/Models/Invoice.php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Invoice extends Model
{
    use HasFactory;

    // If the table name is not the plural of the model name
    protected $table = 'invoices';

    // Defining the relationship with InvoiceDetail
    public function details()
    {
        return $this->hasMany(InvoiceDetail::class, 'invoice_id');
    }
}
Create InvoiceDetail model:

bash
Copy
Edit
php artisan make:model InvoiceDetail
In the InvoiceDetail model, you can define the inverse relationship:

php
Copy
Edit
// app/Models/InvoiceDetail.php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class InvoiceDetail extends Model
{
    use HasFactory;

    // If the table name is not the plural of the model name
    protected $table = 'invoice_details';

    // Defining the inverse relationship with Invoice
    public function invoice()
    {
        return $this->belongsTo(Invoice::class);
    }
}
Step 2: Fetch Data Using Eloquent
Now, in your controller or wherever you're processing the data, you can use the Eloquent models to fetch data from both tables and format it into your desired structure.

Example Controller:
php
Copy
Edit
// app/Http/Controllers/InvoiceController.php
namespace App\Http\Controllers;

use App\Models\Invoice;
use Illuminate\Http\Request;

class InvoiceController extends Controller
{
    public function show($invoiceId)
    {
        // Fetch the invoice along with its details (services)
        $invoice = Invoice::with('details')->find($invoiceId);

        if (!$invoice) {
            return response()->json(['message' => 'Invoice not found'], 404);
        }

        // Structure the data as per your required JSON format
        $invoiceData = [
            'patient_id' => $invoice->patient_id,
            'invoice_total' => $invoice->invoice_total,
            'paid_total' => $invoice->paid_total,
            'payment_term' => $invoice->payment_term,
            'services' => $invoice->details->map(function($detail) {
                return [
                    'id' => $detail->id,
                    'name' => $detail->name,
                    'price' => $detail->price,
                    'unit' => $detail->unit,
                    'discount' => $detail->discount,
                    'vat' => $detail->vat,
                ];
            }),
        ];

        return response()->json($invoiceData);
    }
}
Step 3: Define Your Routes
Now, define the route for displaying the invoice:

php
Copy
Edit
// routes/web.php or routes/api.php
use App\Http\Controllers\InvoiceController;

Route::get('/invoice/{id}', [InvoiceController::class, 'show']);
Explanation:
Eloquent Relationship:
The Invoice model has a hasMany relationship to InvoiceDetail, which means each invoice can have multiple services (details). The InvoiceDetail model has a belongsTo relationship back to the Invoice.

Fetching Data:
The Invoice::with('details') will eagerly load the associated InvoiceDetail records along with the invoice, which is more efficient than doing multiple queries.

Data Transformation:
Using the map() function, you can loop over the details and return a transformed array in the structure you need.

Return JSON:
Finally, we return the data as a JSON response using response()->json(), which is ideal for API responses.

Example Response:
When you access the route /invoice/{id}, you’ll get the data formatted like this:

json
Copy
Edit
{
    "patient_id": "1",
    "invoice_total": "58416",
    "paid_total": "58416",
    "payment_term": "Cash",
    "services": [
        {
            "id": "2",
            "name": "ECG",
            "price": 258,
            "unit": 87,
            "discount": 18,
            "vat": 0
        },
        {
            "id": "1",
            "name": "X-Ray",
            "price": 409,
            "unit": 88,
            "discount": 14,
            "vat": 0
        }
    ]
}
Step 4: Optional - Using SQL Join (If Necessary)
If you prefer to use a SQL join to get the data in a single query (instead of using Eloquent relationships), you can use DB::table() with join().

Here’s how you can do that:

php
Copy
Edit
use Illuminate\Support\Facades\DB;

public function show($invoiceId)
{
    $invoice = DB::table('invoices')
        ->join('invoice_details', 'invoices.id', '=', 'invoice_details.invoice_id')
        ->where('invoices.id', $invoiceId)
        ->get([
            'invoices.patient_id',
            'invoices.invoice_total',
            'invoices.paid_total',
            'invoices.payment_term',
            'invoice_details.id as service_id',
            'invoice_details.name as service_name',
            'invoice_details.price',
            'invoice_details.unit',
            'invoice_details.discount',
            'invoice_details.vat'
        ]);

    if ($invoice->isEmpty()) {
        return response()->json(['message' => 'Invoice not found'], 404);
    }

    // Group data by invoice and format it
    $invoiceData = [
        'patient_id' => $invoice[0]->patient_id,
        'invoice_total' => $invoice[0]->invoice_total,
        'paid_total' => $invoice[0]->paid_total,
        'payment_term' => $invoice[0]->payment_term,
        'services' => $invoice->map(function($service) {
            return [
                'id' => $service->service_id,
                'name' => $service->service_name,
                'price' => $service->price,
                'unit' => $service->unit,
                'discount' => $service->discount,
                'vat' => $service->vat,
            ];
        }),
    ];

    return response()->json($invoiceData);
}
This code fetches the invoice and related services using an SQL join, and then formats the result into the desired JSON structure.