# PHP Examples

Code examples for integrating with the Royalti.io API using PHP.

## Authentication

### JWT Login

```php
function login(string $email, string $password, string $workspaceId): string
{
    // Step 1: Get refresh token
    $ch = curl_init('https://api.royalti.io/auth/login');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        CURLOPT_POSTFIELDS => json_encode([
            'email' => $email,
            'password' => $password,
        ]),
    ]);
    $loginData = json_decode(curl_exec($ch), true);
    curl_close($ch);

    $refreshToken = $loginData['refresh_token'];

    // Step 2: Exchange for access token
    $url = "https://api.royalti.io/auth/authtoken?currentWorkspace={$workspaceId}";
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ["Authorization: Bearer {$refreshToken}"],
    ]);
    $tokenData = json_decode(curl_exec($ch), true);
    curl_close($ch);

    return $tokenData['data']['access_token'];
}
```

### API Key Client

```php
class RoyaltiClient
{
    private string $baseUrl = 'https://api.royalti.io';
    private string $apiKey;

    public function __construct(string $apiKey)
    {
        $this->apiKey = $apiKey;
    }

    public function get(string $path, array $params = []): array
    {
        $url = $this->baseUrl . $path;
        if ($params) {
            $url .= '?' . http_build_query($params);
        }

        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => $this->headers(),
        ]);
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $data = json_decode($response, true);
        if ($httpCode >= 400) {
            throw new \RuntimeException($data['error']['message'] ?? 'Request failed');
        }
        return $data;
    }

    public function post(string $path, array $body = []): array
    {
        return $this->request('POST', $path, $body);
    }

    public function put(string $path, array $body): array
    {
        return $this->request('PUT', $path, $body);
    }

    public function delete(string $path): array
    {
        return $this->request('DELETE', $path);
    }

    private function request(string $method, string $path, ?array $body = null): array
    {
        $ch = curl_init($this->baseUrl . $path);
        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => $this->headers(),
        ]);
        if ($body !== null) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body));
        }
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $data = json_decode($response, true);
        if ($httpCode >= 400) {
            throw new \RuntimeException($data['error']['message'] ?? 'Request failed');
        }
        return $data;
    }

    private function headers(): array
    {
        return [
            "Authorization: Bearer {$this->apiKey}",
            'Content-Type: application/json',
        ];
    }
}
```

## Common Operations

### List Artists (Paginated)

```php
$client = new RoyaltiClient('RWAK_your_key');

$artists = $client->get('/artist/', [
    'page' => 1,
    'size' => 25,
    'sort' => 'artistName',
    'order' => 'asc',
    'search' => 'drake',
]);

echo "Found {$artists['totalItems']} artists\n";
foreach ($artists['data'] as $artist) {
    echo $artist['artistName'] . "\n";
}
```

### Paginate Through All Results

```php
function paginateAll(RoyaltiClient $client, string $path, array $params = []): \Generator
{
    $params = array_merge($params, ['page' => 1, 'size' => 100]);

    do {
        $result = $client->get($path, $params);
        foreach ($result['data'] as $item) {
            yield $item;
        }
        $params['page']++;
    } while ($params['page'] <= $result['totalPages']);
}

// Usage
foreach (paginateAll($client, '/artist/') as $artist) {
    echo $artist['artistName'] . "\n";
}
```

### Create an Artist

```php
$artist = $client->post('/artist/', [
    'artistName' => 'New Artist',
    'label' => 'My Label',
    'genres' => ['Hip-Hop', 'R&B'],
    'links' => [
        'spotify' => 'https://open.spotify.com/artist/...',
        'instagram' => 'https://instagram.com/newartist',
    ],
]);

echo "Created artist: {$artist['data']['id']}\n";
```

### Create a Split

```php
$split = $client->post('/split/', [
    'entity_type' => 'asset',
    'entity_id' => 'asset-uuid-here',
    'split_type' => 'conditional',
    'effective_date' => '2025-01-01',
    'shares' => [
        ['user' => 'user-uuid-1', 'percentage' => 60],
        ['user' => 'user-uuid-2', 'percentage' => 40],
    ],
    'conditions' => [
        'territories' => ['US', 'GB', 'CA'],
        'mode' => 'include',
        'sources' => ['Spotify', 'Apple Music'],
    ],
]);
```

### Fetch Royalty Analytics

```php
$analytics = $client->get('/royalty/dsp', [
    'start' => '2025-01-01',
    'end' => '2025-12-31',
    'includePreviousPeriod' => true,
]);

foreach ($analytics['data'] as $row) {
    printf("%s: $%.2f (%d streams)\n", $row['dsp'], $row['Royalty'], $row['Streams']);
}
```

## File Upload and Processing

```php
function uploadRoyaltyFile(RoyaltiClient $client, string $filePath): array
{
    $fileName = basename($filePath);

    // 1. Get presigned upload URL
    $uploadInfo = $client->get('/file/upload-url', [
        'fileName' => $fileName,
        'contentType' => 'text/csv',
    ])['data'];

    // 2. Upload to cloud storage
    $ch = curl_init($uploadInfo['url']);
    curl_setopt_array($ch, [
        CURLOPT_PUT => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ['Content-Type: text/csv'],
        CURLOPT_INFILE => fopen($filePath, 'r'),
        CURLOPT_INFILESIZE => filesize($filePath),
    ]);
    curl_exec($ch);
    curl_close($ch);

    // 3. Confirm upload
    $confirmation = $client->post('/file/confirm-upload-completion', [
        'fileId' => $uploadInfo['fileId'],
    ])['data'];

    // 4. Poll detection
    $sessionId = $confirmation['sessionId'];
    do {
        sleep(2);
        $detection = $client->get("/file/detection/{$sessionId}");
    } while ($detection['data']['status'] === 'pending');

    // 5. Confirm detection
    $client->post("/file/confirm-detection/{$sessionId}", [
        'confirmed' => true,
    ]);

    return $confirmation;
}
```

## Webhook Receiver (Laravel)

```php
// routes/api.php
Route::post('/webhooks/royalti', [WebhookController::class, 'handle']);

// app/Http/Controllers/WebhookController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class WebhookController extends Controller
{
    public function handle(Request $request)
    {
        // Verify HMAC signature for financial/split events
        $signature = $request->header('X-Royalti-Signature');
        if ($signature) {
            $expected = hash_hmac('sha256', $request->getContent(), config('services.royalti.webhook_secret'));
            if (!hash_equals($expected, $signature)) {
                return response()->json(['error' => 'Invalid signature'], 401);
            }
        }

        $event = $request->input('event');
        $data = $request->input('data');

        match ($event) {
            'PAYMENT_COMPLETED' => logger()->info("Payment \${$data['attributes']['amount']} to {$data['actor']['name']}"),
            'ROYALTY_FILE_PROCESSED' => logger()->info("File processed: {$data['resource']['displayName']}"),
            'ASSET_CREATED' => logger()->info("New track: {$data['resource']['displayName']}"),
            default => null,
        };

        return response()->json(['received' => true]);
    }
}
```

## Webhook Receiver (Plain PHP)

```php
<?php
$secret = getenv('ROYALTI_WEBHOOK_SECRET');
$payload = file_get_contents('php://input');
$data = json_decode($payload, true);

// Verify HMAC signature
$signature = $_SERVER['HTTP_X_ROYALTI_SIGNATURE'] ?? null;
if ($signature) {
    $expected = hash_hmac('sha256', $payload, $secret);
    if (!hash_equals($expected, $signature)) {
        http_response_code(401);
        echo json_encode(['error' => 'Invalid signature']);
        exit;
    }
}

switch ($data['event']) {
    case 'PAYMENT_COMPLETED':
        error_log("Payment \${$data['data']['attributes']['amount']}");
        break;
    case 'ROYALTY_FILE_PROCESSED':
        error_log("File processed: {$data['data']['resource']['displayName']}");
        break;
}

http_response_code(200);
echo json_encode(['received' => true]);
```
