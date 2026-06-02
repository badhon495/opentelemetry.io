---
title: ফ্রন্টএন্ড
cSpell:ignore: typeof
default_lang_commit: 8dad29e2443b7c8739f3be322e5d5eec3baf999f
---

ফ্রন্টএন্ড ব্যবহারকারীদের জন্য একটি UI প্রদান করার পাশাপাশি UI বা অন্যান্য ক্লায়েন্ট দ্বারা ব্যবহৃত একটি API প্রদান করার জন্য দায়িত্বশীল। অ্যাপ্লিকেশনটি একটি React ওয়েব-ভিত্তিক UI এবং API রাউট প্রদানের জন্য [Next.JS](https://nextjs.org/) ভিত্তিক।

[ফ্রন্টএন্ড সোর্স](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/frontend/)

## সার্ভার ইন্সট্রুমেন্টেশন {#server-instrumentation}

আপনার Node.js অ্যাপ্লিকেশন শুরু করার সময় SDK এবং অটো-ইন্সট্রুমেন্টেশন ইনিশিয়ালাইজ করতে একটি Node required module ব্যবহার করা প্রস্তাবিত। OpenTelemetry Node.js SDK ইনিশিয়ালাইজ করার সময়, আপনি ঐচ্ছিকভাবে কোন অটো-ইন্সট্রুমেন্টেশন লাইব্রেরিগুলো ব্যবহার করবেন তা উল্লেখ করতে পারেন, অথবা `getNodeAutoInstrumentations()` ফাংশন ব্যবহার করতে পারেন যা বেশিরভাগ জনপ্রিয় ফ্রেমওয়ার্ক অন্তর্ভুক্ত করে। `utils/telemetry/Instrumentation.js` ফাইলে OTLP এক্সপোর্ট, রিসোর্স অ্যাট্রিবিউট এবং সার্ভিস নামের জন্য স্ট্যান্ডার্ড [OpenTelemetry পরিবেশ ভেরিয়েবল](/docs/specs/otel/configuration/sdk-environment-variables/) এর উপর ভিত্তি করে SDK এবং অটো-ইন্সট্রুমেন্টেশন ইনিশিয়ালাইজ করার জন্য প্রয়োজনীয় সমস্ত কোড রয়েছে।

```javascript
const FrontendTracer = async () => {
  const { ZoneContextManager } = await import('@opentelemetry/context-zone');

  let resource = resourceFromAttributes({
    [ATTR_SERVICE_NAME]: NEXT_PUBLIC_OTEL_SERVICE_NAME,
  });
  const detectedResources = detectResources({ detectors: [browserDetector] });
  resource = resource.merge(detectedResources);

  const provider = new WebTracerProvider({
    resource,
    spanProcessors: [
      new SessionIdProcessor(),
      new BatchSpanProcessor(
        new OTLPTraceExporter({
          url:
            NEXT_PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT ||
            'http://localhost:4318/v1/traces',
        }),
        {
          scheduledDelayMillis: 500,
        },
      ),
    ],
  });

  const contextManager = new ZoneContextManager();

  provider.register({
    contextManager,
    propagator: new CompositePropagator({
      propagators: [
        new W3CBaggagePropagator(),
        new W3CTraceContextPropagator(),
      ],
    }),
  });

  registerInstrumentations({
    tracerProvider: provider,
    instrumentations: [
      getWebAutoInstrumentations({
        '@opentelemetry/instrumentation-fetch': {
          propagateTraceHeaderCorsUrls: /.*/,
          clearTimingResources: true,
          applyCustomAttributesOnSpan(span) {
            span.setAttribute('app.synthetic_request', IS_SYNTHETIC_REQUEST);
          },
        },
      }),
    ],
  });
};
```

Node required module গুলো `--require` কমান্ড লাইন আর্গুমেন্ট ব্যবহার করে লোড করা হয়। এটি `package.json` এর `scripts.start` সেকশনে করা যেতে পারে এবং `npm start` ব্যবহার করে অ্যাপ্লিকেশন শুরু করা যেতে পারে।

```json
"scripts": {
  "start": "node --require ./Instrumentation.js server.js",
},
```

## ট্রেস {#traces}

### স্প্যান এক্সেপশন এবং স্ট্যাটাস {#span-exceptions-and-status}

একটি হ্যান্ডেল করা ত্রুটির পূর্ণ স্ট্যাক ট্রেস সহ একটি স্প্যান ইভেন্ট তৈরি করতে span অবজেক্টের `recordException` ফাংশন ব্যবহার করতে পারেন। একটি এক্সেপশন রেকর্ড করার সময় স্প্যানের স্ট্যাটাস সেই অনুযায়ী সেট করতে ভুলবেন না। এটি `utils/telemetry/InstrumentationMiddleware.ts` ফাইলের `NextApiHandler` ফাংশনের catch ব্লকে দেখা যাবে।

```typescript
span.recordException(error as Exception);
span.setStatus({ code: SpanStatusCode.ERROR });
```

### নতুন স্প্যান তৈরি করা {#create-new-spans}

`Tracer.startSpan("spanName", options)` ব্যবহার করে নতুন স্প্যান তৈরি এবং শুরু করা যায়। স্প্যান কীভাবে তৈরি হবে তা নির্দিষ্ট করতে বেশ কয়েকটি অপশন ব্যবহার করা যায়।

- `root: true` একটি নতুন ট্রেস তৈরি করবে, এই স্প্যানটিকে রুট হিসেবে সেট করবে।
- `links` অন্য স্প্যানের (এমনকি অন্য ট্রেসের মধ্যে) লিঙ্ক নির্দিষ্ট করতে ব্যবহৃত হয় যা রেফারেন্স করা উচিত।
- `attributes` হলো কী/মান জোড়া যা একটি স্প্যানে যোগ করা হয়, সাধারণত অ্যাপ্লিকেশন কনটেক্সটের জন্য ব্যবহৃত হয়।

```typescript
span = tracer.startSpan(`${method}`, {
  root: true,
  kind: SpanKind.SERVER,
  links: [{ context: syntheticSpan.spanContext() }],
  attributes: {
    'app.synthetic_request': true,
    [ATTR_HTTP_RESPONSE_STATUS_CODE]: response.statusCode,
    [ATTR_HTTP_REQUEST_METHOD]: method,
    [ATTR_USER_AGENT_ORIGINAL]: headers['user-agent'] || '',
    [ATTR_URL_PATH]: target,
    [ATTR_URL_FULL]: `${headers.host}${url}`,
    [ATTR_NETWORK_PROTOCOL_VERSION]: httpVersion,
  },
});
```

## ব্রাউজার ইন্সট্রুমেন্টেশন {#browser-instrumentation}

ফ্রন্টএন্ড যে ওয়েব-ভিত্তিক UI প্রদান করে তাও ওয়েব ব্রাউজারের জন্য ইন্সট্রুমেন্ট করা হয়েছে। OpenTelemetry ইন্সট্রুমেন্টেশন `pages/_app.tsx` এ Next.js App কম্পোনেন্টের অংশ হিসেবে অন্তর্ভুক্ত এবং ইনিশিয়ালাইজ করা হয়।

```typescript
import FrontendTracer from '../utils/telemetry/FrontendTracer';

if (typeof window !== 'undefined') FrontendTracer();
```

`utils/telemetry/FrontendTracer.ts` ফাইলে একটি TracerProvider ইনিশিয়ালাইজ করা, একটি OTLP এক্সপোর্ট স্থাপন করা, ট্রেস কনটেক্সট প্রপাগেটর রেজিস্টার করা এবং ওয়েব নির্দিষ্ট অটো-ইন্সট্রুমেন্টেশন লাইব্রেরি রেজিস্টার করার কোড রয়েছে। যেহেতু ব্রাউজার সম্ভবত একটি পৃথক ডোমেনে থাকা একটি OpenTelemetry Collector-এ ডেটা পাঠাবে, তাই CORS হেডারগুলোও সেই অনুযায়ী সেটআপ করা হয়।

ব্যাকএন্ড সার্ভিসগুলোর জন্য `synthetic_request` অ্যাট্রিবিউট ফ্ল্যাগ বহন করার পরিবর্তনের অংশ হিসেবে, `applyCustomAttributesOnSpan` কনফিগারেশন ফাংশন `instrumentation-fetch` লাইব্রেরির কাস্টম স্প্যান অ্যাট্রিবিউট লজিকে যোগ করা হয়েছে যাতে প্রতিটি ব্রাউজার-সাইড স্প্যান এটি অন্তর্ভুক্ত করে।

```typescript
import {
  CompositePropagator,
  W3CBaggagePropagator,
  W3CTraceContextPropagator,
} from '@opentelemetry/core';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { SimpleSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { getWebAutoInstrumentations } from '@opentelemetry/auto-instrumentations-web';
import { resourceFromAttributes } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const FrontendTracer = async () => {
  const { ZoneContextManager } = await import('@opentelemetry/context-zone');

  const provider = new WebTracerProvider({
    resource: resourceFromAttributes({
      [ATTR_SERVICE_NAME]: process.env.NEXT_PUBLIC_OTEL_SERVICE_NAME,
    }),
    spanProcessors: [new SimpleSpanProcessor(new OTLPTraceExporter())],
  });

  const contextManager = new ZoneContextManager();

  provider.register({
    contextManager,
    propagator: new CompositePropagator({
      propagators: [
        new W3CBaggagePropagator(),
        new W3CTraceContextPropagator(),
      ],
    }),
  });

  registerInstrumentations({
    tracerProvider: provider,
    instrumentations: [
      getWebAutoInstrumentations({
        '@opentelemetry/instrumentation-fetch': {
          propagateTraceHeaderCorsUrls: /.*/,
          clearTimingResources: true,
          applyCustomAttributesOnSpan(span) {
            span.setAttribute('app.synthetic_request', 'false');
          },
        },
      }),
    ],
  });
};

export default FrontendTracer;
```

## মেট্রিক্স {#metrics}

TBD

## লগস {#logs}

TBD

## ব্যাগেজ {#baggage}

অনুরোধটি সিন্থেটিক (লোড জেনারেটর থেকে) কিনা তা পরীক্ষা করতে ফ্রন্টএন্ডে OpenTelemetry Baggage ব্যবহার করা হয়। সিন্থেটিক রিকোয়েস্টগুলো একটি নতুন ট্রেস তৈরি করতে বাধ্য করবে। নতুন ট্রেস থেকে রুট স্প্যানে একটি HTTP রিকোয়েস্ট ইন্সট্রুমেন্টেড স্প্যানের মতো অনেক একই অ্যাট্রিবিউট থাকবে।

একটি Baggage আইটেম সেট করা হয়েছে কিনা তা নির্ধারণ করতে, Baggage হেডার পার্স করতে `propagation` API এবং এন্ট্রি পেতে বা সেট করতে `baggage` API ব্যবহার করতে পারেন।

```typescript
const baggage = propagation.getBaggage(context.active());
if (baggage?.getEntry("synthetic_request")?.value == "true") {...}
```
