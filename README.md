# Recaptcha Enterprise for WordPress using AJAX 
## Create token at frontend
use the follwing code into your form
```html
<button type="submit" onclick="handleOnClick(event)">submit</button>
```
```javascript
<script src="https://www.google.com/recaptcha/enterprise.js?render=SITE_KEY"></script>
<script>
    function handleOnClick(e) {
        e.preventDefault();
        grecaptcha.enterprise.ready(async () => {
            const token = await grecaptcha.enterprise.execute('SITE_KEY', {action: 'ACTION_NAME'});
            // console.log(token);
            if (token === '') {
                // do action
            } else {
                $.ajax({
                    url: adminajax.url, // Url to which the request is send
                    type: "POST",             // Type of request to be send, called as method
                    data: {
                        action: 'eventmail',  // wp action name
                        formData: $(".event-form").serialize(), // form data
                        token:token // generated token
                    },
                    success: function (result, status) {
                        $(".event-form")[0].reset();
                        $(".event-form").append(`<p>Form submitted succesfully</p>`);
                        console.log(result);
                    },
                    error: function (err) {
                        $("event-form").append(`<p>Some errors to send the mail.</p>`);
                    }

                });
            }
        });
    }
</script>
```
## Create assesment(Method 1): Using Client Library
### Download Client Library
Use the following command and download the Google client library into your project folder (e.g. theme folder)
```php
composer require google/cloud-recaptcha-enterprise
```
### Include Library in Project
Put this code at the top in functions.php
```php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

require get_template_directory() . '/path/of/autoload'; // e.g. '/vendor/autoload.php'
use Google\Cloud\RecaptchaEnterprise\V1\RecaptchaEnterpriseServiceClient;
use Google\Cloud\RecaptchaEnterprise\V1\Event;
use Google\Cloud\RecaptchaEnterprise\V1\Assessment;
use Google\Cloud\RecaptchaEnterprise\V1\TokenProperties\InvalidReason;
```
### Create Assesment
```php
function create_assessment(
	string $siteKey,
	string $token,
	string $project
): void {
	$service_account_path = get_template_directory() . '/path/of/service-account json file'; // e.g. '/service-account.json'
	putenv('GOOGLE_APPLICATION_CREDENTIALS=' . $service_account_path);
	// TODO: To avoid memory issues, move this client generation outside
	// of this example, and cache it (recommended) or call client.close()
	// before exiting this method.
	$client = new RecaptchaEnterpriseServiceClient();
	$projectName = $client->projectName($project);
	$event = (new Event())
		->setSiteKey($siteKey)
		->setToken($token);

	$assessment = (new Assessment())
		->setEvent($event);

	try {
		$response = $client->createAssessment(
			$projectName,
			$assessment
		);

		// You can use the score only if the assessment is valid,
		// In case of failures like re-submitting the same token, getValid() will return false
		if ($response->getTokenProperties()->getValid() == false) {
			printf('The CreateAssessment() call failed because the token was invalid for the following reason: ');
			printf(InvalidReason::name($response->getTokenProperties()->getInvalidReason()));
		} else {
			printf('The score for the protection action is:');
			printf($response->getRiskAnalysis()->getScore());
			// Optional: You can use the following methods to get more data about the token
			// Action name provided at token generation.
			// printf($response->getTokenProperties()->getAction() . PHP_EOL);
			// The timestamp corresponding to the generation of the token.
			// printf($response->getTokenProperties()->getCreateTime()->getSeconds() . PHP_EOL);
			// The hostname of the page on which the token was generated.
			// printf($response->getTokenProperties()->getHostname() . PHP_EOL);
		}
	} catch (Exception $e) {
		printf('CreateAssessment() call failed with the following error: ');
		printf($e);
	}
}
create_assessment(
 	'SITE_KEY',
 	'TOKEN', // token from fronend
 	'PROJECT_ID'
);
```
## Create assesment(Method 2): Using REST API
use the following code at functions.php
```php
$PROJECT_ID = 'PROJECT_ID';
$API_KEY = 'API_KE';
$SITE_KEY = 'SITE_KEY';
$TOKEN = 'TOKEN'; // generated at frontend
$URL = 'https://recaptchaenterprise.googleapis.com/v1/projects/' .$PROJECT_ID. '/assessments?key=' .$API_KEY;

// Create a new cURL resource
$ch = curl_init();
// Set the URL
curl_setopt($ch, CURLOPT_URL, $URL);
// Set the request type to POST
curl_setopt($ch, CURLOPT_POST, 1);
// Set the JSON data
$jsonData = json_encode(array(
	"event" => array(
		"token" => $TOKEN,
		"siteKey" => $SITE_KEY,
		"expectedAction" => "ACTION_NAME"
	)
));
curl_setopt($ch, CURLOPT_POSTFIELDS, $jsonData);
// Set the headers
$headers = array(
	"Content-Type: application/json",
	"Content-Length: " . strlen($jsonData)
);
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
// Return the response as a string instead of outputting it directly
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
// Execute the cURL request
$response = curl_exec($ch);
// Check for errors and handle the response
if(curl_errno($ch)) {
	echo "cURL Error: " . curl_error($ch);
} else {
	echo "Response: " . $response;
}
// Close the cURL session
curl_close($ch);
```
