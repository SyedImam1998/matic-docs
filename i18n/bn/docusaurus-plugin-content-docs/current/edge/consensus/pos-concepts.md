---
id: pos-concepts
title: প্রুফ অফ স্টেক
description: "প্রুফ অব স্টেক সম্পর্কিত ব্যাখ্যা এবং নির্দেশাবলী।"
keywords:
  - docs
  - polygon
  - edge
  - PoS
  - stake
---

## সংক্ষিপ্ত বিবরণ {#overview}

এই বিভাগটি বর্তমানে Polygon Edge-এর প্রুফ অফ স্টেকে (PoS) উপস্থিত কিছু ধারণার একটি ভালো ওভারভিউ
ওভারভিউ প্রদান করবে বলে আশা করা হচ্ছে।

বিদ্যমান PoA IBFT ইমপ্লিমেন্টেশনের বিকল্প হচ্ছে Polygon Edge-এর প্রুফ অফ স্টেক (PoS) ইমপ্লিমেন্টেশন,
যা নোড অপারেটররা চেইন শুরু করার সময় সহজেই দুটি চেইনের মধ্যে যেকোনো একটি বেছে নিতে পারেন।

## PoS-এর ফিচার {#pos-features}

প্রুফ অফ স্টেক ইমপ্লিমেন্টেশনের মূল লজিকটি
**[স্ট্যাকিং স্মার্ট কন্ট্রাক্ট।](https://github.com/0xPolygon/staking-contracts/blob/main/contracts/Staking.sol)**

একটি PoS মেকানিজমের Polygon Edge চেইনটি চালু করা হলে এই চুক্তিটি আগে থেকেই ডিপ্লয় করা থাকে এবং `0x0000000000000000000000000000000000001001` ঠিকানায় উপলভ্য থাকে
`0` ব্লকের।

### Epochs {#epochs}

Polygon Edge-এ Epochs-কে PoS-এর অতিরিক্ত কনসেপ্ট হিসেবে চালু করা হয়েছে।

Epoch-কে (ব্লকে) একটি বিশেষ সময় কাল হিসেবে বিবেচনা করা হয় যেখানে নির্দিষ্ট কিছু যাচাইকারীর একটি সেট ব্লক নির্মাণ করতে পারে।
তাদের দৈর্ঘ্য পরিবর্তন করা যায়, অর্থাৎ নোড অপারেটররা জেনেসিস জেনারেশনের সময় একটি epoch এর দৈর্ঘ্য কনফিগার করতে পারেন।

প্রতিটি epoch শেষে, একটি _epoch ব্লক_ তৈরি হয় এবং সেই ঘটনার পরে একটি নতুন epoch শুরু হয়। epoch ব্লক সম্পর্কে
আরও জানতে, [Epoch ব্লক](/docs/edge/consensus/pos-concepts#epoch-blocks) বিভাগটি দেখুন।

প্রতিটি epoch এর শেষে যাচাইকারীর সেট আপডেট করা হয়। epoch ব্লক তৈরি করার সময় স্টেকিং স্মার্ট চুক্তি থেকে নোডগুলো
একটি যাচাইকারীর সেটকে কুয়েরি করে এবং প্রাপ্ত ডেটা স্থানীয় স্টোরেজে সংরক্ষণ করে। এই কুয়েরি এবং সংরক্ষণ চক্র প্রতিটি epoch এর শেষে পুনরাবৃত্তি হয়।

মূলত, এটি স্টেকিং স্মার্ট চুক্তির যাচাইকারীর সেটের ঠিকানাগুলোর উপরে সম্পূর্ণ নিয়ন্ত্রণ থাকার বিশয়টি নিশ্চিত করে এবং
তা নোডগুলোকে শুধুমাত্র 1টি দায়িত্ব প্রদান করে - epoch-এর সময় সাম্প্রতিকতম যাচাইকারীর সেটের তথ্য নিয়ে আসার জন্য
চুক্তিটিকে কুয়েরি করার। এটি একক নোডের একটি যাচাইকারীর সেটের দায়িত্ব নেয়ার বোঝা কমিয়ে দেয়।

### স্টেকিং {#staking}

ঠিকানাগুলো `stake` পদ্ধতি ব্যবহার করে এবং লেনদেনে স্টেক করা পরিমাণের একটি মান নির্দিষ্ট করে
স্টেকিং স্মার্ট চুক্তিতে ফান্ড স্টেক করতে পারেন:

````js
const StakingContractFactory = await ethers.getContractFactory("Staking");
let stakingContract = await StakingContractFactory.attach(STAKING_CONTRACT_ADDRESS)
as
Staking;
stakingContract = stakingContract.connect(account);

const tx = await stakingContract.stake({value: STAKE_AMOUNT})
````

স্টেকিং স্মার্ট চুক্তিতে ফান্ড স্টেক করে ঠিকানাগুলো যাচাইকারীর সেটে অন্তর্ভুক্ত হতে পারে এবং ফলশ্রুতিতে
ব্লক তৈরি প্রক্রিয়ায় অংশগ্রহণ করতে পারে।

:::info স্টেকিং-এর থ্রেশহোল্ড

বর্তমানে, যাচাইকারীর সেটে অন্তর্ভুক্ত হওয়ার জন্য সর্বনিম্ন থ্রেশহোল্ড হচ্ছে `1 ETH` স্টেক করা

:::

### আনস্টেকিং {#unstaking}

যেসব ঠিকানা ফান্ড স্টেক করেছে, তারা তাদের **সমস্ত স্টেক করা ফান্ড একত্রে আনস্টেক করতে পারবেন**।

স্টেকিং স্মার্ট চুক্তিতে `unstake` পদ্ধতি কল করে আনস্টেকিং করা যেতে পারে:

````js
const StakingContractFactory = await ethers.getContractFactory("Staking");
let stakingContract = await StakingContractFactory.attach(STAKING_CONTRACT_ADDRESS)
as
Staking;
stakingContract = stakingContract.connect(account);

const tx = await stakingContract.unstake()
````

ফান্ড আনস্টেক করার পরে, স্টেকিং স্মার্ট চুক্তি থেকে ঠিকানাগুলোকে সরিয়ে ফেলা হয় এবং সেগুলোকে
পরবর্তী epoch-এর সময় আর যাচাইকারী হিসেবে গণ্য করা হবে না।

## Epoch ব্লক {#epoch-blocks}

**Epoch ব্লক** Polygon Edge-এ IBFT-এর PoS ইমপ্লিমেন্টেশনের একটি কনসেপ্ট হিসেবে চালু করা হয়েছে।

মূলত, epoch ব্লকগুলো বিশেষ ব্লক যাতে কোন **লেনদেন থাকে না** এবং শুধুমাত্র **epoch-এর শেষ দিকেই** ঘটে। উদাহরণস্বরূপ, যদি **epoch আকার** ব্লক সেট করা `50`হয়, তাহলে epoch ব্লক `50`ব্লক হতে হবে , `100``150`এবং তাই হবে।

তাদের অতিরিক্ত লজিক সম্পাদন করতে ব্যবহার করা হয় যা নিয়মিত ব্লক প্রোডাকশনের সময় ঘটে না।

সবচেয়ে গুরুত্বপূর্ণ বিষয় হচ্ছে, তারা নোডের একটি নির্দেশনা হিসেবে কাজ করে যে **সাম্প্রতিকতম যাচাইকারীর সেটের তথ্য নিয়ে আসতে হবে**
স্টেকিং স্মার্ট চুক্তি থেকে।

Epoch ব্লকে যাচাইকারীর সেট আপডেট করার পর, স্টেকিং স্মার্ট চুক্তি থেকে সর্বশেষ তথ্য নিয়ে এসে আবার আপডেট না হওয়া পর্যন্ত, যাচাইকারীর সেট (পরিবর্তিত বা অপরিবর্তিত যাই হোক না কেন) পরবর্তী `epochSize - 1` ব্লকের জন্য
ব্যবহার করা হয়।

জেনেসিস ফাইল তৈরি করার সময় Epoch এর দৈর্ঘ্য (ব্লকে) একটি বিশেষ ফ্ল্যাগ `--epoch-size` ব্যবহার করে পরিবর্তন করা যায়:

```bash
polygon-edge genesis --epoch-size 50 ...
```

Polygon Edge-এ একটি epoch-এর ডিফল্ট আকার হচ্ছে `100000` ব্লক।

## চুক্তির প্রি-ডিপ্লয়মেন্ট {#contract-pre-deployment}

Polygon Edge _প্রি-ডিপ্লয়_ করে
[স্টেকিং স্মার্ট চুক্তি](https://github.com/0xPolygon/staking-contracts/blob/main/contracts/Staking.sol)-কে
`0x0000000000000000000000000000000000001001` ঠিকানার মধ্যে **জেনেসিস তৈরির** সময়।

এটি কোন চলমান EVM ছাড়াই জেনেসিস কমান্ডে পাস করা কনফিগারেশন ভ্যালু ব্যবহার করে, সরাসরি ব্লকচেইনের স্মার্ট চুক্তির অবস্থা
পরিবর্তনের মাধ্যমে তা করে থাকে।