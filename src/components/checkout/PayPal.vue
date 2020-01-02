<template>
    <div>
        <div ref="paypal"></div>
    </div>
</template>

<script>

import gql from 'graphql-tag';
import CART_FRAGMENT from '../Cart.gql';
import MONEY_FRAGMENT from '../Money.gql';
import ADDRESS_FRAGMENT from '../Address.gql';
import cartMixin from '../../mixins/cartMixin';

export default {
  name: 'PayPal',

  data() {
    return {
      loaded: false,
      paidFor: false,
      paypal: {
        sandbox: 'AXmWwNrV5Fniihxwsi11dG6zarVkjGr8cvxR5h4hi5csTTFEw41fbHa2M0rrO_cirldjuL8MkppRFVIv',
        production: '',
      },
      me: null,
    };
  },

  mixins: [cartMixin],

  apollo: {
    me: {
      query: gql`
        query me($locale: Locale!) {
          me {
            activeCart {
              ...CartFields
            }
          }
        }
        ${CART_FRAGMENT}
        ${MONEY_FRAGMENT}
        ${ADDRESS_FRAGMENT}`,
      variables() {
        return {
          locale: this.$store.state.locale,
        };
      },
    },
  },
  mounted() {
    const script = document.createElement('script');
    script.src = `https://www.paypal.com/sdk/js?currency=EUR&intent=authorize&client-id=${this.paypal.sandbox}`;
    script.addEventListener('load', this.setLoaded);
    document.body.appendChild(script);
  },
  methods: {
    setPaypalPaymentToCart(payment) {
      this.updateMyCart(payment);
    },

    setLoaded() {
      this.loaded = true;
      const cartId = this.me.activeCart.id;
      console.log(cartId);
      window.paypal
        .Buttons({
          createOrder() {
            return fetch(`http://localhost:8000/paypal-connector/cart/${cartId}/authorize`, {
              method: 'get',
              headers: {
                'content-type': 'application/json',
              },
            })
              .then(res => res.text());
          },

          onApprove: async (data, actions) => {
            actions.order.authorize().then((authorization) => {
              const authorizationID = authorization.purchase_units[0].payments.authorizations[0].id;
              console.log(`OrderId:${data.orderID}`);
              console.log(`authorizationID:${authorizationID}`);
              this.setPaypalPaymentToCart([{
                setCustomType: {
                  typeKey: 'PaypalPayment',
                  fields: [
                    {
                      name: 'authorizationId',
                      value: `"${authorizationID.toString()}"`,
                    },
                    {
                      name: 'orderId',
                      value: `"${data.orderID.toString()}"`,
                    }],
                },
              }]);
            });
          },
          onError: (err) => {
            console.log(err);
          },
        })
        .render(this.$refs.paypal);
    },
  },
};
</script>
