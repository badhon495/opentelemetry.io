---
title: React Native অ্যাপ
default_lang_commit: c1f7650c58f7b70efcfd944886b83bcc8e0ff912
---

React Native অ্যাপটি Android এবং iOS ডিভাইসের ব্যবহারকারীদের জন্য একটি মোবাইল UI প্রদান করে, যার মাধ্যমে তারা ডেমোর সার্ভিসগুলোর সাথে ইন্টারঅ্যাক্ট করতে পারেন। এটি [Expo](https://docs.expo.dev/get-started/create-a-project/) দিয়ে তৈরি এবং অ্যাপের স্ক্রিনগুলো সাজানোর জন্য Expo-এর ফাইল-ভিত্তিক রাউটিং ব্যবহার করে।

[React Native অ্যাপ সোর্স](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/react-native-app/)

## ইন্সট্রুমেন্টেশন {#instrumentation}

অ্যাপ্লিকেশনটি JS লেয়ারে অ্যাপ্লিকেশন ইন্সট্রুমেন্ট করতে OpenTelemetry প্যাকেজ ব্যবহার করে।

> [!CAUTION]
>
> JS OTel প্যাকেজগুলো node এবং web পরিবেশের জন্য সমর্থিত। যদিও এগুলো React Native-এর জন্যও কাজ করে, তবে সেই পরিবেশের জন্য এগুলো স্পষ্টভাবে সমর্থিত নয়, যেখানে মাইনর ভার্সন আপডেটে সামঞ্জস্যতা নষ্ট হতে পারে বা workaround প্রয়োজন হতে পারে। React Native-এর জন্য JS OTel প্যাকেজ সমর্থন তৈরি করা সক্রিয় উন্নয়নের একটি ক্ষেত্র।

অ্যাপ্লিকেশনের মূল এন্ট্রি পয়েন্ট হলো `app/_layout.tsx`, যেখানে ইন্সট্রুমেন্টেশন ইনিশিয়ালাইজ করতে এবং UI প্রদর্শনের আগে এটি লোড হয়েছে কিনা তা নিশ্চিত করতে একটি hook ব্যবহার করা হয়:

```typescript
import { useTracer } from '@/hooks/useTracer';

const { loaded: tracerLoaded } = useTracer();
```

`hooks/useTracer.ts`-এ ইন্সট্রুমেন্টেশন সেটআপের সমস্ত কোড রয়েছে, যার মধ্যে একটি TracerProvider ইনিশিয়ালাইজ করা, একটি OTLP এক্সপোর্ট স্থাপন করা, ট্রেস কনটেক্সট প্রপাগেটর রেজিস্টার করা এবং নেটওয়ার্ক রিকোয়েস্টের অটো-ইন্সট্রুমেন্টেশন রেজিস্টার করা অন্তর্ভুক্ত।

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
      // মোবাইল অ্যাপের কনটেক্সটে ইন্সট্রুমেন্টেশন কাজ করাতে propagateTraceHeaderCorsUrls প্রয়োজন,
      // যদিও আমরা CORS রিকোয়েস্ট করছি না। ওয়েব-শুধুমাত্র Performance API এড়াতে
      // `clearTimingResources` বন্ধ রাখতে হবে
      new FetchInstrumentation({
        propagateTraceHeaderCorsUrls: /.*/,
        clearTimingResources: false,
      }),

      // React Native-এ fetch-এর ইম্পলিমেন্টেশন আসলে XMLHttpRequest-এর উপরে একটি polyfill:
      // https://github.com/facebook/react-native/blob/7ccc5934d0f341f9bc8157f18913a7b340f5db2d/packages/react-native/Libraries/Network/fetch.js#L17
      // এ কারণে `fetch` দিয়ে রিকোয়েস্ট করলে XMLHttpRequest দিয়ে করা অন্তর্নিহিত রিকোয়েস্টের জন্য
      // একটি অতিরিক্ত span তৈরি হয়। যেহেতু এই ডেমোতে /api/ তে কলগুলো fetch ব্যবহার করে,
      // সেই পথে অতিরিক্ত span এড়াতে XMLHttpRequest ইন্সট্রুমেন্টেশন বন্ধ করুন।
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
