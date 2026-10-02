# Migración de Stripe a Mercado Pago

Esta guía es una lista de verificación para rediseñar tu integración de pagos en Node.js; no es una equivalencia uno a uno entre las API de Stripe y Mercado Pago. Este paquete exporta clientes de Mercado Pago, no las API del SDK de Stripe. Los objetos de Stripe, métodos de pago guardados y suscripciones no se migran automáticamente: planifica cómo tratar los datos y acuerdos existentes con cada proveedor.

## Lista de verificación

1. **Sustituye credenciales e instalación.** Crea una cuenta e integración de Mercado Pago, obtiene tus credenciales de prueba y producción y guarda el `accessToken` del servidor de forma segura. Instala el SDK con `npm install --save mercadopago`; deja de usar las claves y el SDK de Stripe en los nuevos flujos de Mercado Pago. Consulta la [instalación](README.md#-installation).
2. **Selecciona el flujo de cobro.** Decide según el país, los medios de pago y la experiencia deseada si usarás [Orders API](https://mercadopago.com/developers/es/docs/order/landing) para pagos en tu sitio o un checkout con redirección. No asumas que un PaymentIntent, Checkout Session o suscripción de Stripe tiene un sustituto directo. Revisa los requisitos y campos del flujo elegido antes de cambiar tus formularios.
3. **Adapta la creación de pagos.** Configura `MercadoPagoConfig` con `accessToken`, instancia `Order` y envía el cuerpo de la orden con `order.create({ body, requestOptions })`. El importe de la orden y el de la transacción son cadenas decimales; ajusta moneda, medio de pago y datos del pagador según el flujo. Este ejemplo corresponde a una orden con tarjeta tokenizada, no a todos los flujos:

   ```js
   import { MercadoPagoConfig, Order } from 'mercadopago';

   const client = new MercadoPagoConfig({ accessToken: process.env.MP_ACCESS_TOKEN });
   const order = new Order(client);
   const body = {
     type: 'online',
     processing_mode: 'automatic',
     total_amount: '1000.00',
     external_reference: 'pedido-123',
     payer: { email: 'comprador@example.com' },
     transactions: {
       payments: [{
         amount: '1000.00',
         payment_method: {
           id: 'master',
           type: 'credit_card',
           token: '<CARD_TOKEN>',
           installments: 1,
         },
       }],
     },
   };
   const requestOptions = { idempotencyKey: '<CLAVE_UNICA_POR_OPERACION>' };
   const result = await order.create({ body, requestOptions });
   ```

   El `idempotencyKey` pertenece a `requestOptions` (o a las opciones globales de configuración), no al `body`. Usa una clave nueva por operación y reutilízala al reintentar **la misma** operación; conserva la referencia a tu pedido para conciliar resultados. Consulta el [ejemplo de creación de órdenes](src/examples/order/create.ts) y la [guía de inicio](README.md#-getting-started).
4. **Revisa la tokenización de tarjetas.** No reutilices tokens de Stripe ni envíes datos de tarjeta sin un flujo compatible; obtén un token de tarjeta válido para Mercado Pago mediante su flujo de tokenización y colócalo en `transactions.payments[].payment_method.token` solo si tu flujo de órdenes usa tarjeta. Analiza por separado cómo pedir consentimiento o volver a recopilar medios de pago guardados.
5. **Ajusta estados y notificaciones.** Reemplaza la lógica que depende de eventos y estados de Stripe por el ciclo de vida de la orden y sus pagos de Mercado Pago. Procesa notificaciones de forma segura, verifica la firma cuando corresponda, consulta el estado actual del recurso antes de confirmar el pedido y contempla pagos pendientes, aprobados, rechazados, cancelaciones y reembolsos. No supongas que la respuesta inicial equivale a un cobro definitivo ni que los webhooks de ambos proveedores son intercambiables.
6. **Valida antes del cambio de proveedor.** En un entorno de prueba, comprueba credenciales, tokenización, flujos aprobados y rechazados, idempotencia ante reintentos, notificaciones duplicadas o demoradas, conciliación y reembolsos. Define qué pasará con los pagos y suscripciones ya activos en Stripe, monitorea el nuevo flujo y solo entonces habilita gradualmente las credenciales de producción.