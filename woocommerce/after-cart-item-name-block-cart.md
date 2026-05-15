# Replicate after_cart_item_name_hook in block cart

The shortcode block provides the `woocommerce_after_cart_item_name` hook to add content after each product name in the cart, eg:

```php
add_action( 'woocommerce_after_cart_item_name', 'add_hello_world_after_cart_item_name', 10, 2 );

function add_hello_world_after_cart_item_name( $cart_item, $cart_item_key ) {
    echo '<div>Hello world</div>';
}
```

This can be replicated for the block cart using a JS observer until more extensibility is added to cart block (mentioned in https://github.com/woocommerce/woocommerce/discussions/62636#discussioncomment-15723530). eg:

```js
document.addEventListener( 'DOMContentLoaded', function () {
	const cartBlock = document.querySelector( '.wp-block-woocommerce-cart' );

	if ( ! cartBlock ) {
		return;
	}

	const observer = new MutationObserver( ( mutations ) => {
		for ( const mutation of mutations ) {
			for ( const node of mutation.addedNodes ) {
				if ( ! ( node instanceof HTMLElement ) ) {
					continue;
				}
				const row = node.matches( '.wc-block-cart-items__row' );
				if ( ! row ) {
					continue;
				}
				const prices = node.querySelector( '.wc-block-cart-item__prices' );
				if ( ! prices ) {
					continue;
				}
				prices.insertAdjacentHTML(
					'afterend',
					'<div>Hello world</div>'
				);
			}
		}
	} );

	observer.observe( cartBlock, {
		childList: true,
		subtree: true,
	} );
} );
```

TODO: Combine this with [cartItemClass filter](https://developer.woocommerce.com/docs/block-development/extensible-blocks/cart-and-checkout-blocks/filters-in-cart-and-checkout/cart-line-items/) to conditionally add content.
