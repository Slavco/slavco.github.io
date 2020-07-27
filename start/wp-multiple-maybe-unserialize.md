# WordPress and multiple maybe_unserialize 

WordPress introduced `maybe_unserialize` and `maybe_serialize` as security functions in the past and they are deeply integrated in the core. Usually for backward compatibility or improper usage in the past, many software vendors are using multiple `maybe_unserialize` towards values that could be planted into DB with lower number of `maybe_serialize`.

## Eli5 PoC

```
$maybe_double_unserialize = function ( $value ) {
	return maybe_unserialize( $value );
};

$values = array_map(
	$maybe_double_unserialize,
	array_filter( get_post_meta( $product_id, $field_key ) )
);
```
and if you think that this code isn't existing then check this out. Yes it is 100k active installs Woo plugin :/

## Few facts

- dounble `maybe_unserialize` is quite widespread around WP eco system, specially in Woo related plugins


## Remadiation

- add constraints into your php.ini regarding serialization
- mod the maybe_ functions so they sign the final payload

