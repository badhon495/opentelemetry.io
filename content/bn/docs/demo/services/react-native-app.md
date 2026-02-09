---
title: React Native অ্যাপ
default_lang_commit: ae417344d183999236c22834435e0dfeb109da29
---

React Native অ্যাপটি Android এবং iOS ডিভাইসে ব্যবহারকারীদের জন্য একটি মোবাইল UI প্রদান করে যাতে তারা ডেমোর সার্ভিসগুলোর সাথে ইন্টারঅ্যাক্ট করতে পারে। এটি [Expo](https://docs.expo.dev/get-started/introduction/) দিয়ে তৈরি এবং অ্যাপের স্ক্রিন লেআউট করার জন্য Expo এর ফাইল-ভিত্তিক রাউটিং ব্যবহার করে।

[React Native অ্যাপ সোর্স](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/react-native-app/)

## ইন্সট্রুমেন্টেশন

অ্যাপ্লিকেশনটি JS লেয়ারে অ্যাপ্লিকেশন ইন্সট্রুমেন্ট করার জন্য OpenTelemetry প্যাকেজগুলি ব্যবহার করে।

> [!CAUTION]
>
> JS OTel প্যাকেজগুলি node এবং web এনভায়রনমেন্টের জন্য সাপোর্টেড। যদিও তারা React Native এর জন্যও কাজ করে, কিন্তু সেই এনভায়রনমেন্টের জন্য স্পষ্টভাবে সাপোর্টেড নয়, যেখানে তারা মাইনর ভার্সন আপডেটের সাথে সামঞ্জস্য ভেঙে ফেলতে পারে বা workaround প্রয়োজন হতে পারে। React Native এর জন্য JS OTel প্যাকেজ সাপোর্ট তৈরি করা সক্রিয় উন্নয়নের একটি ক্ষেত্র।

অ্যাপ্লিকেশনের মূল এন্ট্রি পয়েন্ট হলো `app/_layout.tsx` যেখানে ইন্সট্রুমেন্টেশন ইনিশিয়ালাইজ করতে এবং UI প্রদর্শনের আগে এটি লোড হওয়া নিশ্চিত করতে একটি hook ব্যবহার করা হয়:

```typescript
import { useTracer } from '@/hooks/useTracer';

const { loaded: tracerLoaded } = useTracer();
```

`hooks/useTracer.ts` এ ইন্সট্রুমেন্টেশন সেট আপ করার জন্য সমস্ত কোড রয়েছে, যার মধ্যে একটি TracerProvider ইনিশিয়ালাইজ করা, একটি OTLP এক্সপোর্ট স্থাপন করা, trace context propagators রেজিস্টার করা এবং নেটওয়ার্ক রিকোয়েস্টের অটো-ইন্সট্রুমেন্টেশন রেজিস্টার করা অন্তর্ভুক্ত।

```typescript
import {
  CompositePropagator,
  W3CBaggagePropagator,
  W3CTraceContextPropagator,
} from '@opentelemetry/core';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { XMLHttpRequestInstrumentation } from '@opentelemetry/instrumentation-xml-http-request';
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { resourceFromAttributes } from '@opentelemetry/resources';
import {
  ATTR_DEVICE_ID,
  ATTR_OS_NAME,
  ATTR_OS_VERSION,
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
} from '@opentelemetry/semantic-conventions/incubating';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import getLocalhost from '@/utils/Localhost';
import { useEffect, useState } from 'react';
import {
  getDeviceId,
  getSystemVersion,
  getVersion,
} from 'react-native-device-info';
import { Platform } from 'react-native';
import { SessionIdProcessor } from '@/utils/SessionIdProcessor';

const Tracer = async () => {
  const localhost = await getLocalhost();

  const resource = resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'react-native-app',
    [ATTR_OS_NAME]: Platform.OS,
    [ATTR_OS_VERSION]: getSystemVersion(),
    [ATTR_SERVICE_VERSION]: getVersion(),
    [ATTR_DEVICE_ID]: getDeviceId(),
  });

  const provider = new WebTracerProvider({
    resource,
    spanProcessors: [
      new BatchSpanProcessor(
        new OTLPTraceExporter({
          url: `http://${localhost}:${process.env.EXPO_PUBLIC_FRONTEND_PROXY_PORT}/otlp-http/v1/traces`,
        }),
        {
          scheduledDelayMillis: 500,
        },
      ),
      new SessionIdProcessor(),
    ],
  });

  provider.register({
    propagator: new CompositePropagator({
      propagators: [
        new W3CBaggagePropagator(),
        new W3CTraceContextPropagator(),
      ],
    }),
  });

  registerInstrumentations({
    instrumentations: [
      // Some tiptoeing required here, propagateTraceHeaderCorsUrls is required to make the instrumentation
      // work in the context of a mobile app even though we are not making CORS requests. `clearTimingResources` must
      // be turned off to avoid using the web-only Performance API
      new FetchInstrumentation({
        propagateTraceHeaderCorsUrls: /.*/,
        clearTimingResources: false,
      }),

      // The React Native implementation of fetch is simply a polyfill on top of XMLHttpRequest:
      // https://github.com/facebook/react-native/blob/7ccc5934d0f341f9bc8157f18913a7b340f5db2d/packages/react-native/Libraries/Network/fetch.js#L17
      // Because of this when making requests using `fetch` there will an additional span created for the underlying
      // request made with XMLHttpRequest. Since in this demo calls to /api/ are made using fetch, turn off
      // instrumentation for that path to avoid the extra spans.
      new XMLHttpRequestInstrumentation({
        ignoreUrls: [/\/api\/.*/],
      }),
    ],
  });
};

export interface TracerResult {
  loaded: boolean;
}

export const useTracer = (): TracerResult => {
  const [loaded, setLoaded] = useState<boolean>(false);

  useEffect(() => {
    if (!loaded) {
      Tracer()
        .catch(() => console.warn('failed to setup tracer'))
        .finally(() => setLoaded(true));
    }
  }, [loaded]);

  return {
    loaded,
  };
};
```
