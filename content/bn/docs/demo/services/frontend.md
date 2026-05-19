---
title: ফ্রন্টএন্ড
cSpell:ignore: typeof
default_lang_commit: 8dad29e2443b7c8739f3be322e5d5eec3baf999f
---

ফ্রন্টএন্ড, ব্যবহারকারীদের জন্য একটি UI প্রদান করে, পাশাপাশি UI বা অন্যান্য ক্লায়েন্ট দ্বারা ব্যবহৃত একটি API প্রদান করে। এই অ্যাপ্লিকেশনটি [Next.JS](https://nextjs.org/) এর উপর ভিত্তি করে তৈরি যা একটি React ওয়েব-ভিত্তিক UI এবং API রুট প্রদান করে।

[ফ্রন্টএন্ড সোর্স](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/frontend/)

## সার্ভার ইন্সট্রুমেন্টেশন {#server-instrumentation}

Node.js অ্যাপ্লিকেশন শুরু করার সময়, SDK এবং অটো-ইন্সট্রুমেন্টেশন ইনিশিয়ালাইজ করার জন্য Node required module ব্যবহার করা রিকমান্ডেড। OpenTelemetry Node.js SDK ইনিশিয়ালাইজ করার সময়, আপনি ঐচ্ছিকভাবে উল্লেখ করতে পারেন কোন অটো-ইন্সট্রুমেন্টেশন লাইব্রেরি ব্যবহার করবেন, অথবা `getNodeAutoInstrumentations()` ফাংশন ব্যবহার করতে পারেন যা সবচেয়ে জনপ্রিয় ফ্রেমওয়ার্কগুলো অন্তর্ভুক্ত করে। `utils/telemetry/Instrumentation.js` ফাইলে OTLP এক্সপোর্ট, রিসোর্স অ্যাট্রিবিউট এবং সার্ভিস নামের জন্য স্ট্যান্ডার্ড [OpenTelemetry এনভায়রনমেন্ট ভেরিয়েবল](/docs/specs/otel/configuration/sdk-environment-variables/) এর উপর ভিত্তি করে SDK এবং অটো-ইন্সট্রুমেন্টেশন ইনিশিয়ালাইজ করার জন্য প্রয়োজনীয় সমস্ত কোড রয়েছে।

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

## ট্রেস (#traces)

### স্প্যান এক্সসেপশন এবং স্ট্যাটাস (#span-exception-and-status)

একটি হ্যান্ডল করা ভুল (error) এর সম্পূর্ণ স্ট্যাক ট্রেস সহ একটি span event তৈরি করতে আপনি span অবজেক্টের `recordException` ফাংশন ব্যবহার করতে পারেন। exception রেকর্ড করার সময় span এর status যথাযথভাবে সেট করতে ভুলবেন না। আপনি এটি `utils/telemetry/InstrumentationMiddleware.ts` ফাইলের `NextApiHandler` ফাংশনের catch ব্লকে দেখতে পারেন।

```typescript
span.recordException(error as Exception);
span.setStatus({ code: SpanStatusCode.ERROR });
```

### নতুন স্প্যান তৈরি করা (#create-new-spans)

নতুন span তৈরি এবং শুরু করার জন্য `Tracer.startSpan("spanName", options)` ব্যবহার করা যায়। span কীভাবে তৈরি করা হবে তা উল্লেখ করার জন্য বেশ কিছু options ব্যবহার করা যেতে পারে।

- `root: true` একটি নতুন trace তৈরি করবে, এই span কে root হিসেবে সেট করবে।
- `links` অন্যান্য span এর (এমনকি অন্য trace এর মধ্যেও) লিঙ্ক উল্লেখ করতে ব্যবহার করা হয়, যা রেফারেন্স করা উচিত।
- `attributes` হলো span এ যোগ করা key/value জোড়া, সাধারণত অ্যাপ্লিকেশন কনটেক্সটের জন্য ব্যবহৃত হয়।

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

ফ্রন্টএন্ড যে ওয়েব-ভিত্তিক UI প্রদান করে তা ওয়েব ব্রাউজারের জন্যও ইন্সট্রুমেন্ট করা। OpenTelemetry ইন্সট্রুমেন্টেশন `pages/_app.tsx` এ Next.js App কম্পোনেন্টের অংশ হিসেবে অন্তর্ভুক্ত। এখানে ইন্সট্রুমেন্টেশন ইমপোর্ট এবং ইনিশিয়ালাইজ করা হয়।

```typescript
import FrontendTracer from '../utils/telemetry/FrontendTracer';

if (typeof window !== 'undefined') FrontendTracer();
```

`utils/telemetry/FrontendTracer.ts` ফাইলে TracerProvider ইনিশিয়ালাইজ করার, OTLP এক্সপোর্ট স্থাপন করার, trace context propagators রেজিস্টার করার এবং ওয়েব স্পেসিফিক অটো-ইন্সট্রুমেন্টেশন লাইব্রেরি রেজিস্টার করার কোড রয়েছে। যেহেতু ব্রাউজার একটি OpenTelemetry Collector এ ডেটা পাঠাবে যা সম্ভবত আলাদা ডোমেইনে থাকবে, তাই CORS হেডার ও সেই অনুযায়ী সেটআপ করা হয়েছে।

পরিবর্তনের অংশ হিসেবে, ব্যাকএন্ড সার্ভিসের জন্য `synthetic_request` অ্যাট্রিবিউট ফ্ল্যাগ বহন করার, `applyCustomAttributesOnSpan` কনফিগারেশন ফাংশন `instrumentation-fetch` লাইব্রেরির কাস্টম span attributes লজিকে যোগ করা হয়েছে যাতে প্রতিটি ব্রাউজার-সাইড span এটি অন্তর্ভুক্ত করে।

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

## মেট্রিক্স

TBD

## লগস

TBD

## ব্যাগেজ

OpenTelemetry Baggage ফ্রন্টএন্ডে ব্যবহার করা হয় যাচাই করার জন্য যে রিকোয়েস্টটি synthetic কিনা (লোড জেনারেটর থেকে)। Synthetic রিকোয়েস্ট একটি নতুন trace তৈরি করতে বাধ্য করবে। নতুন trace এর root span এ একটি HTTP রিকোয়েস্ট ইন্সট্রুমেন্ট করা span এর মতো একই অ্যাট্রিবিউট থাকবে।

একটি Baggage আইটেম সেট আছে কিনা তা নির্ধারণ করতে, আপনি Baggage হেডার পার্স করতে `propagation` API ব্যবহার করতে পারেন এবং এন্ট্রি পেতে বা সেট করতে `baggage` API ব্যবহার করতে পারেন।

```typescript
const baggage = propagation.getBaggage(context.active());
if (baggage?.getEntry("synthetic_request")?.value == "true") {...}
```
