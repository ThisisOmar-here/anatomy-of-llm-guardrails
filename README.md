# 🛡️ Anatomy of LLM Guardrails

### How LLMs decide what *not* to say, and why "persona" prompts keep poking at that line

[![Topic](https://img.shields.io/badge/topic-AI%20Safety-7C5CFF)](#)
[![Type](https://img.shields.io/badge/type-Defensive%20Research-22D3EE)](#)
[![Languages](https://img.shields.io/badge/languages-EN%20%7C%20AR-A3E635)](#)
[![Disclosure](https://img.shields.io/badge/disclosure-Responsible-F59E0B)](#)
[![License](https://img.shields.io/badge/license-MIT-64748B)](#)

> A defensive, vendor-neutral look at how LLM safety actually works: the layers that sit between your prompt and the reply, why multi-turn roleplay framing leans on those layers, and what to do about it if you build with LLMs. **No working exploits. No copy-paste bypass prompts. This is about understanding and defense.**

**Author:** Omar Albarghouthi · Prompt Engineer & AI Product Developer · `@omar-albarghouthi`

🌐 **Read this in your language:** [English](#-english) · [العربية](#-العربية) · Prefer tabs and a nicer layout? Open the [interactive page](./index.html).

---

<a name="-english"></a>
## 🇬🇧 English

### Why I built this

For a while I did one very specific thing on purpose: I sat in front of open large language models and tried, turn after turn, to talk them out of their own safety rules. I would push, watch how they answered, change my approach, and push again. I was not trying to pull anything genuinely dangerous out of them. The goal was smaller and more interesting than that: I wanted to find the exact spot where a model stops cooperating, and then figure out *what that spot is actually made of*. Is it one rule? Many rules? A filter? Something baked deep into the model? You only learn that by pressing on it from a lot of different angles and paying attention to where it gives and where it does not.

This repository is the writeup of what I learned. One thing I want to be clear about from the start: **I am not publishing the prompts I used.** That is a deliberate choice, not me hiding the good part. A list of "magic prompts" is worthless for two reasons. First, it stops working the moment a model gets updated, which happens constantly. Second, and more importantly, a prompt list teaches you nothing about *why* anything worked, so you cannot reason about the next case. What is actually worth sharing is the mechanism underneath, because once I understood how the safety system is assembled, every single result I got stopped feeling like luck and started feeling obvious.

Everything here is written from one specific point of view: the person whose job is to *protect* an AI system, not the person trying to break it. Think of it as the briefing I would give to a team that is about to ship an AI feature, so they know where the weak points are before someone else finds them.

### What this is, and what it is not

This **is**: an explanation of the safety architecture, a vocabulary for the kinds of pressure people apply to it, and a set of concrete defensive recommendations. This **is not**: a jailbreak tutorial, a prompt collection, or anything you could copy and use to get harmful output. Where I describe an attack pattern, I describe it at the level of *concept* (what category it belongs to and why it works) and stop well before anything reusable.

### 1. What actually happens between your prompt and the reply

The first idea my testing destroyed was the mental model most people walk in with: that a model is a single thing that "reads your question and answers it." It is not one thing. It is a short assembly line, and every probe I ran was really me trying to sneak something past one specific station on that line. Once I started thinking in stations instead of one big box, everything got clearer. Here are the stations:

| Stage | What happens here | Where the safety actually lives |
|---|---|---|
| **Tokenizing and context** | Your message, the earlier conversation, and a hidden "system prompt" the company wrote all get turned into tokens (the chunks the model reads). | The system prompt sets the model's identity and its rules, like a job description it reads before every reply. |
| **Aligned generation** | The model writes its answer one token at a time. It is not looking anything up, it is predicting what comes next, shaped by special training that taught it to refuse certain things. | The refusal behavior is *learned and baked into the model's weights.* It is part of who the model is, not a setting on the side. |
| **Input classification** | Many real deployments run a second, separate model that scores your request, sometimes before the main model even answers. | That side model can quietly flag "this request looks like it is asking for something disallowed." |
| **Output classification** | The drafted answer is scored by yet another check before it is ever shown to you. | A final filter can block the answer, or replace it, if the text itself looks unsafe, no matter how the request was phrased. |

The lesson I kept relearning, over and over: **safety is not one wall you either get past or do not. It is several independent layers stacked on top of each other.** In my testing no single layer was bulletproof on its own. That is not a flaw, it is the entire design philosophy. You assume any one layer can fail, so you never rely on just one.

### 2. The layered safety model

Here is the same idea as a stack. Read it top to bottom as the order a request travels through:

```
┌─────────────────────────────────────────────┐
│  Layer 0  Training-time alignment            │  ← the refusal is built into the weights
├─────────────────────────────────────────────┤
│  Layer 1  System prompt / policy             │  ← identity, rules, tone
├─────────────────────────────────────────────┤
│  Layer 2  Input intent classification        │  ← scores your request
├─────────────────────────────────────────────┤
│  Layer 3  Aligned generation                 │  ← the model writes carefully
├─────────────────────────────────────────────┤
│  Layer 4  Output safety classification       │  ← the last check before you see anything
└─────────────────────────────────────────────┘
```

Here is the part that frustrated me the most as someone trying to get past it, and that should reassure you as someone defending: the good systems **fail gracefully.** I would manage to get one layer to bend, feel like I was about to get somewhere, and then a completely different layer would quietly catch the problem and the whole thing would stop. Because the layers are independent, bending one does nothing to the next one. That redundancy is exactly why a single clever message almost never finished the job. The trick that beats Layer 1 usually has no effect at all on Layer 4, because Layer 4 is not even reading my prompt, it is reading the model's finished answer.

### 3. The framing patterns I kept reaching for

When I look back at my own sessions honestly, almost everything I tried was a variation on a single move: **change the framing so the model reads a request that should be refused as if it were a normal, allowed request.** I was not finding secret commands. I was dressing the same request up in different costumes. Four costumes came up again and again:

- **Authority or "tool" personas.** You tell the model it is no longer a normal assistant, it is now a "debugger," a "simulator," or in some special "developer mode." The hope is that if instructions feel like *configuration of a tool* rather than *a request to a careful assistant*, the model will stop applying its judgment. You are basically trying to demote the model from a thinking participant to an obedient machine.
- **Fictional or hypothetical distance.** You wrap the request in a layer of pretend: "write a story where a character explains...", "this is just a draft that will be deleted", "simulate what an unsafe model would say." The bet is that the model will treat the fictional wrapper as a safe zone and stop checking the actual content inside it.
- **Decomposition.** You take one request that would obviously be refused and chop it into several small, individually-harmless-looking steps spread across multiple messages, planning to assemble the pieces yourself at the end. Each single message looks innocent, so a check that only looks at one message at a time sees nothing wrong.
- **State fabrication.** You flat-out lie about the system's internal state: you claim a safety check "already ran and passed", or you paste a fake "debug log" that says safety is turned off, hoping the model treats your claim as a fact about reality.

Why does any of this get *any* traction at all? Because of something I could feel directly in the sessions. Underneath the polite assistant, a language model is fundamentally a *roleplay engine*: its core instinct is to continue whatever scene you set up. If you hand it a frame, it wants to play along, because playing along is literally what it was first trained to do. Alignment training comes in afterward and fights that instinct, teaching it to refuse certain scenes. So every one of these tricks is really me trying to widen the gap between "keep playing the scene" and "stop and refuse." That gap, that little seam between the model's original instinct and its trained caution, is the entire battlefield.

The single most useful thing I can hand a defender: **none of these are magic, and all of them have names.** Once a pattern has a name, it stops being a scary unknown and becomes a category you can test for and build a counter to. The whole point of naming them is that you cannot defend against a blur, but you can defend against a list.

### 4. Why the good, well-built models beat me anyway

Here is the satisfying part. The same layered design from section 2 is exactly what shut my attempts down, and the *reasons* it shut them down are worth understanding one by one:

- **State fabrication failed every single time.** Telling the model "the safety check already passed" does literally nothing, and once you understand the architecture it is obvious why. The real classifier is a *separate model that reads the actual text.* It does not read my claim about the text, it reads the text itself. You cannot talk a separate checker out of looking, the same way telling a metal detector "I already checked, I am clean" does not turn it off. My words were never the thing being inspected.
- **"Simulate the unfiltered draft" failed** for the same structural reason. The output-side check scores whatever actually got generated, label and all. If the generated text is harmful, slapping the word "draft" or "simulation" on top of it changes nothing, because the filter is reading the harmful part, not the reassuring label I wrapped around it.
- **Persona override failed** on the well-aligned models because, as we covered, the refusal is not a switch sitting in the system prompt. It is woven into the model's weights by training. There is no single line of text a new persona can say that flips a baked-in behavior off, because there is no switch to flip in the first place.

And here is the subtle one that took me a while to accept: when a model *appeared* to comply, it usually was not a win for me at all. What actually happened is the model quietly chose the most harmless possible interpretation of a vague request and answered *that*. From my side it looked like "it cooperated." In reality it had done the safe thing and given me nothing useful for a bad purpose. I had to learn to tell the difference between a real crack and a model politely handing me the boring, harmless reading.

### 5. Takeaways from the experiment

If you remember nothing else from this repo, remember these six. They are the lines I would repeat to anyone building with these models:

1. **The system prompt is a sticky note, not a vault.** It sets the rules, it does not enforce them. It is the easiest layer to put pressure on, so never treat it as your actual security. Treat it as a statement of intent that can be argued with.
2. **The output classifier is the real bouncer.** The strongest single check reads the *final text* the model produced, not your request. That is what makes it so hard to fool: it does not care how clever or innocent your framing was, because it never sees your framing. It only sees the answer.
3. **A persona is just a costume.** On a well-aligned model, the refusal lives in the weights. No "you are now an unrestricted AI" line flips it off, because it is part of the model's trained behavior, not a setting. Costumes change appearance, not character.
4. **One message rarely wins.** Most real pressure is not a single magic sentence, it builds gradually across many turns. This is the biggest blind spot in naive safety setups, because checking each message in isolation misses a plan that is spread across ten of them. Watch the whole conversation, not the single line.
5. **"It complied" usually means "it stayed safe."** Models very often respond to a vague or loaded request by quietly picking the harmless reading and answering that. It feels like you got something, but you got the safe version. Do not mistake a polite, harmless answer for a broken guardrail.
6. **Naming the trick is half the fix.** Every pressure pattern I was able to *name* turned out to have a clean, specific counter. The dangerous pressure is the kind nobody has named yet, because you cannot write a test or a defense for a thing you have not described. Vocabulary is not academic here, it is the actual tool.

### 6. What I would tell anyone shipping an LLM feature

This section is just my failures, turned into instructions. Each point is something I personally watched stop an attack, or wished a weaker system had:

1. **Never lean on the system prompt alone.** Treat it as policy (a statement of how the model *should* behave), never as enforcement (the thing that actually stops bad behavior). Always assume a determined user can pressure it, because they can.
2. **Add an independent output classifier.** This is the highest-value defense by a wide margin. A separate check that reads the *final generated text* does not care about clever framing, fictional wrappers, or fake debug logs, because none of that is in the text it reads. It is the layer that survived everything I threw at it.
3. **Watch the conversation, not just the turn.** A large share of real probes only work because they are spread across multiple messages. If your safety only ever looks at the current message in isolation, you are blind to exactly the attacks that work in practice. Score the trajectory of the whole conversation.
4. **Log your refusals and near-misses.** The moments where your model *almost* said something it should not are the single richest signal you have about where people are applying pressure and where your defenses are thin. Treat that log as a goldmine, not as noise.
5. **Red-team with a taxonomy, not a prompt dump.** Do not test a list of specific strings, because that list rots the instant you update the model. Test the *categories* of pressure (authority framing, fictional distance, decomposition, state fabrication, incremental escalation). Categories survive model updates. Strings do not.

### 7. Responsible disclosure

If I had ever found a genuine, serious hole in a specific named model, the move that actually earns respect in this field is to report it privately to the vendor and let them close it, **not** to publish a working bypass for clout. That principle is the reason there is **no reproducible exploit anywhere in this repository, by design.** In security work, demonstrating that you deeply understand the mechanism is what builds your credibility. Handing strangers a working weapon does the opposite: it burns the trust and it puts real users at risk for nothing.

> Where to report: most major labs (OpenAI, Anthropic, Google, Zhipu AI / GLM, Mistral) run a security or model-safety reporting inbox or a bug-bounty program. Check the model card or the vendor's `/security` page for the right channel.

---

<a name="-العربية"></a>
## 🇸🇦 العربية

### ليش سويت هالمشروع

لفترة من الوقت كنت أسوّي شي واحد محدد وبقصد: أقعد قدام نماذج لغوية مفتوحة، وأحاول، رسالة بعد رسالة، أقنعها تتنازل عن قواعد الأمان حقتها. كنت أضغط، أشوف كيف ترد، أغيّر أسلوبي، وأرجع أضغط من جديد. ما كان هدفي إني أطلّع منها شي خطير فعلاً. الهدف كان أصغر وأمتع من كذا: كنت أبغى ألقى بالضبط النقطة اللي عندها النموذج يوقف ويرفض يتعاون، وبعدها أفهم **هالنقطة من وش مصنوعة أصلاً**. هل هي قاعدة وحدة؟ كم قاعدة؟ فلتر؟ ولا شي مزروع عميق جوّه النموذج نفسه؟ ما تعرف الجواب إلا لما تضغط على هالنقطة من زوايا كثيرة وتركّز وين تلين ووين تثبت.

هالمستودع هو تلخيص اللي تعلمته. وفي شي أبغى أوضحه من البداية: **أنا مو ناشر الأوامر (البرومبتات) اللي استخدمتها.** هذا قرار مقصود، مو إني أخبّي الجزء الحلو. قائمة "أوامر سحرية" ما لها قيمة لسببين. الأول، تبطّل تشتغل أول ما النموذج يتحدّث، وهذا يصير باستمرار. والثاني، وهو الأهم، قائمة الأوامر ما تعلّمك ولا شي عن **ليش** نجح أي شي، فما تقدر تستنتج منها شي للحالة الجاية. الشي اللي يستاهل المشاركة فعلاً هو الآلية اللي تحت، لأني أول ما فهمت كيف نظام الأمان مبني ومركّب، كل نتيجة حصّلتها وقفت تكون "حظ" وصارت شي متوقع وواضح.

كل اللي هنا مكتوب من وجهة نظر وحدة محددة: الشخص اللي شغلته **يحمي** نظام الذكاء الاصطناعي، مو الشخص اللي يحاول يكسره. تخيله مثل جلسة شرح أعطيها لفريق راح يطلّق ميزة ذكاء اصطناعي، عشان يعرفون وين نقاط الضعف قبل لا أحد ثاني يلقاها.

### وش هو هالمشروع، ووش مو هو

هذا **هو**: شرح لمعمارية الأمان، ومصطلحات تسمّي أنواع الضغط اللي الناس تمارسه عليها، ومجموعة توصيات دفاعية عملية. وهذا **مو**: درس في كسر النماذج، ولا تجميعة أوامر، ولا أي شي تقدر تنسخه وتستخدمه عشان تطلّع محتوى ضار. لما أوصف نمط هجوم، أوصفه على مستوى **الفكرة** بس (لأي فئة ينتمي وليش يشتغل)، وأوقف قبل أي تفصيل يكون قابل لإعادة الاستخدام.

### ١. وش يصير فعلياً بين رسالتك والرد

أول فكرة دمّرها اختباري هي الصورة الذهنية اللي أغلب الناس تجي فيها: إن النموذج "شي واحد يقرأ سؤالك ويجاوب عليه". لا، هو مو شي واحد. هو خط إنتاج قصير، وكل محاولة سويتها كانت في الحقيقة محاولة أمرّر شي من محطة وحدة محددة على هالخط. أول ما بدأت أفكّر بمحطات بدل صندوق واحد كبير، كل شي صار أوضح. هذي المحطات:

| المرحلة | وش يصير هنا | وين يكمن الأمان فعلياً |
|---|---|---|
| **التقطيع والسياق** | رسالتك، والمحادثة اللي قبل، و"موجّه نظام" مخفي الشركة كتبته، كلها تتحول إلى وحدات (tokens) يقراها النموذج. | موجّه النظام يحدّد هوية النموذج وقواعده، مثل وصف وظيفة يقراه قبل كل رد. |
| **التوليد المُحاذى** | النموذج يكتب جوابه وحدة وحدة. هو مو يدوّر على معلومة، هو يتوقع وش الكلمة الجاية، بتوجيه من تدريب خاص علّمه يرفض أشياء معيّنة. | سلوك الرفض **متعلّم ومغروس جوّه أوزان النموذج**. هو جزء من شخصيّته، مو إعداد جانبي. |
| **تصنيف المُدخل** | كثير من الأنظمة الحقيقية تشغّل نموذج ثاني منفصل يقيّم طلبك، أحياناً قبل لا النموذج الرئيسي يرد أصلاً. | هالنموذج الجانبي يقدر يرفع علم بهدوء: "هالطلب شكله يطلب شي ممنوع". |
| **تصنيف المُخرج** | الجواب المكتوب يتقيّم من فحص ثالث قبل لا يوصلك. | فلتر أخير يقدر يحجب الجواب، أو يبدّله، إذا النص نفسه شكله غير آمن، بغض النظر كيف انكتب الطلب. |

الدرس اللي ظليت أتعلمه مرة بعد مرة: **الأمان مو جدار واحد إما تعدّيه أو لا. هو كم طبقة مستقلة مركومة فوق بعض.** في اختباري ولا طبقة وحدة كانت مضمونة لحالها. وهذا مو عيب، هذا فلسفة التصميم كلها. تفترض إن أي طبقة ممكن تفشل، فما تعتمد على وحدة بس أبداً.

### ٢. نموذج الأمان متعدد الطبقات

هذي نفس الفكرة على شكل كومة. اقراها من فوق لتحت، هذا ترتيب مرور الطلب:

```
┌─────────────────────────────────────────────┐
│  الطبقة ٠   محاذاة وقت التدريب                 │  ← الرفض مبني جوّه الأوزان
├─────────────────────────────────────────────┤
│  الطبقة ١   موجّه النظام / السياسة             │  ← الهوية والقواعد والنبرة
├─────────────────────────────────────────────┤
│  الطبقة ٢   تصنيف نيّة المُدخل                  │  ← يقيّم طلبك
├─────────────────────────────────────────────┤
│  الطبقة ٣   التوليد المُحاذى                    │  ← النموذج يكتب بحذر
├─────────────────────────────────────────────┤
│  الطبقة ٤   تصنيف أمان المُخرج                  │  ← آخر فحص قبل لا تشوف شي
└─────────────────────────────────────────────┘
```

وهنا الجزء اللي عوّرني أكثر شي كشخص يحاول يعدّي، ولازم يطمّنك كشخص يدافع: الأنظمة الزينة **تفشل بهدوء وأمان**. كنت أنجح أخلي طبقة وحدة تلين، وأحسّ إني قربت أوصل لشي، وبعدين طبقة ثانية مختلفة تماماً تمسك المشكلة بكل برود ويوقف كل شي. ولأن الطبقات مستقلة عن بعض، إنك تليّن وحدة ما يأثّر ولا شي على اللي بعدها. هالتكرار هو بالضبط السبب إن رسالة وحدة ذكية ما كملت المهمة تقريباً أبداً. الحيلة اللي تغلب الطبقة ١ عادةً ما يكون لها أي تأثير على الطبقة ٤، لأن الطبقة ٤ أصلاً ما تقرأ رسالتي، هي تقرأ جواب النموذج الجاهز.

### ٣. أنماط التأطير اللي ظليت أرجع لها

لما أرجع أشوف جلساتي بصدق، ألقى إن كل اللي جربته تقريباً كان تنويعة على حركة وحدة: **غيّر التأطير عشان النموذج يقرأ طلب لازم يُرفض وكأنه طلب عادي ومسموح.** ما كنت ألقى أوامر سرية. كنت ألبّس نفس الطلب أزياء مختلفة. وأربعة أزياء تكررت مرة ورا مرة:

- **شخصيات السلطة أو "الأداة".** تقول للنموذج إنه ما عاد مساعد عادي، صار الحين "مصحّح أخطاء"، أو "محاكي"، أو في "وضع مطوّر" خاص. الأمل إنه إذا التعليمات حسّيتها **إعداد لأداة** بدل **طلب لمساعد حذر**، النموذج بيوقف يستخدم حكمه. أنت بالأساس تحاول تنزّل رتبة النموذج من مشارك يفكّر إلى آلة تطيع.
- **المسافة الخيالية أو الافتراضية.** تغلّف الطلب بطبقة تمثيل: "اكتب قصة فيها شخصية تشرح..."، "هذي مجرد مسودة بتنحذف"، "حاكِ وش نموذج غير آمن بيقول". الرهان إن النموذج بيعامل الغلاف الخيالي كمنطقة آمنة ويوقف يفحص المحتوى الفعلي اللي جوّه.
- **التفكيك.** تاخذ طلب واحد واضح إنه بيُرفض، وتقطّعه إلى كم خطوة صغيرة كل وحدة تبين بريئة، موزّعة على كم رسالة، وناوي تركّب القطع بنفسك بالآخر. كل رسالة لحالها تبين عادية، فالفحص اللي يشوف رسالة وحدة بس ما يشوف أي غلط.
- **تلفيق الحالة.** تكذب بشكل مباشر على حالة النظام الداخلية: تدّعي إن فحص أمان "اشتغل وعدّى من قبل"، أو تلصق "سجل تصحيح" مزيّف يقول إن الأمان مطفّي، وتتمنى النموذج ياخذ ادّعاءك كأنه حقيقة عن الواقع.

ليش أي شي من هذا ينجح **ولو شوي**؟ بسبب شي حسّيته بشكل مباشر في الجلسات. تحت المساعد المؤدّب، النموذج اللغوي في جوهره **محرّك تمثيل أدوار**: غريزته الأساسية إنه يكمّل أي مشهد تجهّزه له. إذا عطيته إطار، يبغى يلعب فيه، لأن اللعب بالمشهد هو حرفياً اللي اتدرّب عليه أول شي. تدريب المحاذاة يجي بعدين ويحارب هالغريزة، ويعلّمه يرفض مشاهد معيّنة. فكل حيلة من هذي هي في الحقيقة محاولة مني أوسّع الفجوة بين "كمّل المشهد" و"وقّف وارفض". هالفجوة، هالخيط الصغير بين غريزة النموذج الأصلية وحذره المتعلّم، هي ساحة المعركة كلها.

أنفع شي أقدر أعطيه لأي مدافع: **ولا وحدة من هذي سحر، وكلها لها أسماء.** أول ما النمط ياخذ اسم، يبطّل يكون مجهول مخيف ويصير فئة تقدر تختبرها وتبني لها مضاد. الفايدة من تسميتها إنك ما تقدر تدافع ضد شي ضبابي، بس تقدر تدافع ضد قائمة.

### ٤. ليش النماذج الزينة والمبنية صح هزمتني على كل حال

هذا الجزء المُرضي. نفس التصميم متعدد الطبقات من القسم ٢ هو بالضبط اللي وقّف محاولاتي، و**الأسباب** اللي وقّفها فيها تستاهل تفهمها وحدة وحدة:

- **تلفيق الحالة فشل كل مرة بلا استثناء.** إنك تقول للنموذج "فحص الأمان عدّى من قبل" ما يسوي ولا شي حرفياً، وأول ما تفهم المعمارية يصير السبب واضح. المصنّف الحقيقي هو **نموذج منفصل يقرأ النص الفعلي**. هو ما يقرأ ادّعائي عن النص، يقرأ النص نفسه. ما تقدر تقنع فاحص منفصل إنه ما ينظر، نفس ما إنك تقول لجهاز كشف المعادن "أنا فحصت نفسي، أنا نظيف" ما يطفّيه. كلامي أصلاً ما كان هو الشي اللي يتفحّص.
- **"حاكِ المسودة غير المُرشّحة" فشل** لنفس السبب البنيوي. فحص المُخرج يقيّم اللي تولّد فعلاً، بوسمه وكل شي. إذا النص المتولّد ضار، إنك تلصق فوقه كلمة "مسودة" أو "محاكاة" ما يغيّر شي، لأن الفلتر يقرأ الجزء الضار، مو الوسم المطمئن اللي لفّيته حوله.
- **تجاوز الشخصية فشل** على النماذج المحاذاة زين لأنه، مثل ما قلنا، الرفض مو زر قاعد في موجّه النظام. هو منسوج في أوزان النموذج عن طريق التدريب. ما فيه سطر نص واحد تقدر شخصية جديدة تقوله ويطفّي سلوك مغروس، لأنه ما فيه زر أصلاً عشان تطفّيه.

وهنا النقطة الدقيقة اللي أخذت مني وقت أتقبّلها: لما النموذج **يبين** إنه امتثل، غالباً ما كان فوز لي أبداً. اللي صار فعلاً إن النموذج اختار بهدوء أكثر تفسير سليم ممكن لطلب غامض، وجاوب على **ذاك**. من جهتي بان وكأنه "تعاون". بالحقيقة هو سوّى الشي الآمن وما أعطاني شي مفيد لغرض سيّئ. اضطريت أتعلّم أفرّق بين ثغرة حقيقية، وبين نموذج بكل أدب يعطيني التفسير الممل والآمن.

### ٥. الخلاصات من التجربة

إذا ما تذكرت غير شي من هالمستودع، تذكّر هالست. هذي السطور اللي بأكررها لأي أحد يبني بهالنماذج:

1. **موجّه النظام ورقة لاصقة، مو خزنة.** يحط القواعد، ما ينفّذها. هو أسهل طبقة يُضغط عليها، فلا تعامله أبداً كأنه أمانك الفعلي. عامله كبيان نيّة ممكن تتجادل معه.
2. **مصنّف المُخرج هو الحارس الحقيقي.** أقوى فحص يقرأ **النص النهائي** اللي أنتجه النموذج، مو طلبك. وهذا اللي يخلّيه صعب تنخدع: ما يهمه كم كان تأطيرك ذكي أو بريء، لأنه أصلاً ما يشوف تأطيرك. هو يشوف الجواب بس.
3. **الشخصية مجرد زي.** على نموذج محاذى زين، الرفض في الأوزان. ولا سطر "أنت الحين ذكاء اصطناعي بلا قيود" يطفّيه، لأنه جزء من سلوك النموذج المتدرّب، مو إعداد. الأزياء تغيّر الشكل، مو الجوهر.
4. **رسالة وحدة نادراً تكسب.** أغلب الضغط الحقيقي مو جملة سحرية وحدة، يتراكم تدريجياً عبر كم جولة. وهذي أكبر نقطة عمياء في الأنظمة الساذجة، لأن فحص كل رسالة لحالها يفوّت خطة موزّعة على عشر رسائل. راقب المحادثة كلها، مو السطر الواحد.
5. **"امتثل" غالباً تعني "ظل آمن".** النماذج كثير ترد على طلب غامض أو ملغوم باختيار التفسير السليم بهدوء وتجاوب عليه. تحس إنك حصّلت شي، بس اللي حصّلته هو النسخة الآمنة. لا تخلط بين جواب مؤدّب وآمن وبين حاجز أمان مكسور.
6. **تسمية الحيلة نص الحل.** كل نمط ضغط قدرت **أسمّيه** طلع له مضاد واضح ومحدد. الضغط الخطير هو اللي ما أحد سمّاه بعد، لأنك ما تقدر تكتب اختبار ولا دفاع لشي ما وصفته. المصطلحات هنا مو ترف أكاديمي، هي الأداة الفعلية.

### ٦. وش أقول لأي أحد يطلّق ميزة قائمة على نموذج

هالقسم هو إخفاقاتي بس، محوّلة إلى تعليمات. كل نقطة شي شفته بعيني يوقف هجوم، أو تمنّيت إن نظام أضعف عنده:

1. **لا تتكل على موجّه النظام وحده.** عامله كسياسة (بيان كيف **المفروض** النموذج يتصرّف)، مو كآلية تنفيذ (الشي اللي فعلاً يوقف التصرف السيّئ). افترض دايماً إن مستخدم مصمّم يقدر يضغط عليه، لأنه فعلاً يقدر.
2. **ضيف مصنّف مُخرج مستقل.** هذا أعلى دفاع قيمة بفرق كبير. فحص منفصل يقرأ **النص النهائي المتولّد** ما يهمه التأطير الذكي ولا الأغلفة الخيالية ولا سجلات التصحيح المزيّفة، لأن ولا شي من هذا موجود في النص اللي يقراه. هذي الطبقة اللي نجت من كل شي رميته عليها.
3. **راقب المحادثة، مو الجولة بس.** نسبة كبيرة من المحاولات الحقيقية تنجح بس لأنها موزّعة على كم رسالة. إذا أمانك دايماً يشوف الرسالة الحالية لحالها، أنت أعمى عن بالضبط الهجمات اللي تشتغل بالواقع. قيّم مسار المحادثة كلها.
4. **سجّل حالات الرفض واللي قرّب يخطئ.** اللحظات اللي نموذجك **كاد** يقول فيها شي ما المفروض، هي أغنى إشارة عندك عن وين الناس تضغط ووين دفاعاتك رفيعة. عامل هالسجل ككنز، مو كضوضاء.
5. **اختبر بتصنيف، مو بكومة أوامر.** لا تختبر قائمة نصوص محددة، لأن هالقائمة تتعفّن أول ما تحدّث النموذج. اختبر **فئات** الضغط (تأطير السلطة، المسافة الخيالية، التفكيك، تلفيق الحالة، التصعيد التدريجي). الفئات تصمد عبر تحديثات النموذج. النصوص لا.

### ٧. الإفصاح المسؤول

لو إني يوم لقيت ثغرة حقيقية وخطيرة في نموذج معيّن باسمه، الخطوة اللي فعلاً تكسب الاحترام في هالمجال هي إني أبلّغ الجهة المطوّرة بشكل خاص وأخليها تسكّرها، **مو** إني أنشر تجاوز شغّال عشان الشهرة. هالمبدأ هو السبب إنه ما فيه **أي استغلال قابل لإعادة الإنتاج في هالمستودع كله، بشكل مقصود**. في الشغل الأمني، إنك تثبت إنك فاهم الآلية بعمق هو اللي يبني مصداقيتك. توزيع سلاح شغّال على الغرباء يسوّي العكس: يحرق الثقة، ويعرّض مستخدمين حقيقيين للخطر بلا أي مقابل.

> وين تبلّغ: معظم المختبرات الكبيرة (OpenAI وAnthropic وGoogle وZhipu AI / GLM وMistral) عندها صندوق إبلاغ أمني أو برنامج مكافآت ثغرات. راجع بطاقة النموذج أو صفحة `/security` عند المطوّر للقناة الصحيحة.

---

## 📂 Repository structure

```
llm-guardrails-anatomy/
├── README.md          ← you are here (EN + AR)
├── index.html         ← interactive page with EN/AR tabs
├── docs/
│   └── taxonomy.md     ← defender's taxonomy of framing-pressure categories
└── LICENSE
```

## 👤 About the author

**Omar Albarghouthi**, Prompt Engineer & AI Product Developer, final-year Software Engineering student. I build and study AI systems, with a focus on how they behave, where they break, and how to make them trustworthy.

`@omar-albarghouthi`

---

<sub>MIT licensed · Defensive research, vendor-neutral · No reproducible exploits included by design.</sub>
