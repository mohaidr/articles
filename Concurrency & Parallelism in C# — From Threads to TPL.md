# مفاهيم أساسية في البرمجة المتوازية والتزامنية في C#

---

## 1. الـ Process والـ Thread

تذكر أن الـ **Process** هو التطبيق، والـ **Thread** هو مسار تنفيذ داخله.

**مثال الكود:**

```csharp
using System;
using System.Diagnostics;
using System.Threading;

// الـ Process هو هذا البرنامج عند تشغيله
class Program {
    static void Main() {
        // إنشاء Thread (عامل) جديد داخل هذا الـ Process
        Thread worker = new Thread(() => {
            Console.WriteLine("أنا خيط (Thread) أعمل داخل هذا المصنع!");
        });
        
        worker.Start();
    }
}
```

---

## 2. الـ Multi-threading والـ Asynchronous

- ال **Multi-threading (تعدد الخيوط):** تشغيل كودين في نفس اللحظة فعلياً عن طريق إنشاء خيوط (Threads) إضافية.
- ال **Async (الانتظار الذكي):** تحرير الـ Thread الحالي من الانتظار أثناء جلب البيانات، بدون إنشاء thread جديد.

### مثال مقارنة كاملة 
```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        Console.WriteLine($"[Main] يعمل على Thread: {Thread.CurrentThread.ManagedThreadId}");

        // ───────────────────────────────────────────
        // الطريقة 1: Multi-threading
        // ───────────────────────────────────────────
        // نفتح Thread جديد كلياً من الـ OS — له تكلفة إنشاء وذاكرة خاصة (~1MB stack)
        // الـ Main لن يتجمد، لكننا "أنفقنا" thread حقيقي طول فترة الانتظار
        Thread t = new Thread(() =>
        {
            Console.WriteLine($"[Thread] بدأت على Thread: {Thread.CurrentThread.ManagedThreadId}");
            Thread.Sleep(2000); // محاكاة عمل ثقيل أو انتظار شبكة
            Console.WriteLine($"[Thread] انتهيت على Thread: {Thread.CurrentThread.ManagedThreadId}");
        });
        t.Start();

        Console.WriteLine("[Main] تابع عمله بينما Thread يشتغل في الخلفية...");
        t.Join(); // انتظر الـ thread ينتهي

        Console.WriteLine();

        // ───────────────────────────────────────────
        // الطريقة 2: Async / Await
        // ───────────────────────────────────────────
        // لا يوجد thread جديد! نفس thread الـ Main يُحرَّر ويعمل على أشياء أخرى
        // بينما الـ OS ينتظر رد الشبكة — لما يجي الرد يكمل من حيث وقف
        Console.WriteLine($"[Main] قبل await — Thread: {Thread.CurrentThread.ManagedThreadId}");
        var loadDataTask = LoadDataAsync();
        
        Console.WriteLine($"[Main] Do Work While Data Loading");

        await loadDataTask;
        Console.WriteLine($"[Main] بعد await — Thread: {Thread.CurrentThread.ManagedThreadId}");
    }

    static async Task LoadDataAsync()
    {
        Console.WriteLine($"  [Async] بدأت الطلب على Thread: {Thread.CurrentThread.ManagedThreadId}");

        using var client = new HttpClient();
        // عند الـ await: الـ thread يُحرَّر ويرجع لـ thread pool — لا أحد ينتظر!
        string data = await client.GetStringAsync("https://jsonplaceholder.typicode.com/todos/1");

        // بعد رجوع الرد، يكمل على thread من الـ pool (قد يكون نفسه أو غيره)
        Console.WriteLine($"  [Async] انتهيت على Thread: {Thread.CurrentThread.ManagedThreadId}");
        Console.WriteLine($"  [Async] أول 50 حرف من البيانات: {data[..50]}...");
    }
}
```

### لماذا Async أفضل من Multi-threading لعمليات الشبكة والـ I/O؟

ال **Multi-threading:**
- الـ Threads المستخدمة: thread جديد من الـ OS لكل عملية
- استهلاك الذاكرة: ~1MB stack لكل thread
- 1000 طلب متزامن: 1000 thread = كارثة!
- مناسب لـ: عمليات CPU الثقيلة

ال **Async/Await:**
- الـ Threads المستخدمة: نفس الـ thread، يُحرَّر أثناء الانتظار
- استهلاك الذاكرة: لا شيء تقريباً
- 1000 طلب متزامن: يُدار بعدد صغير من threads
- مناسب لـ: شبكة، قاعدة بيانات، ملفات

> **الخلاصة:** كلاهما لا يجمّد الـ Main، لكن الـ async أكثر كفاءة لأنه لا "يهدر" thread حقيقي طول فترة الانتظار. تخيّل 10,000 طلب HTTP — مع threads ستحتاج 10,000 thread، مع async يكفيك عشرات فقط.

---

## 3. الـ Concurrency والـ Parallelism

- ال **Concurrency:** التبديل السريع بين المهام (كالأم التي تهتم بشؤون عائلتها فتغسل وتطبخ وتجلي وترعى الأطفال وتنتقل بين أعمالها — وكأنها تنجزها جميعاً معاً، لكنها في الحقيقة تتنقل بينهم).
- ال **Parallelism:** تنفيذ المهام معاً في نفس اللحظة (كأنك تطبخ وصديقك يغسل الأطباق).

### مثال Concurrency — مهام متداخلة على thread واحد أو عدد محدود

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class ConcurrencyExample
{
    static async Task Main()
    {
        Console.WriteLine("=== Concurrency: نبدأ 3 مهام معاً بدون انتظار كل واحدة ===\n");

        // نطلق الثلاث مهام دفعة واحدة — لا ننتظر كل واحدة تخلص قبل التالية
        // هذا هو الـ Concurrency: التنسيق بين مهام متداخلة زمنياً
        Task task1 = SimulateWork("طلب API", delaySeconds: 3);
        Task task2 = SimulateWork("قراءة ملف", delaySeconds: 1);
        Task task3 = SimulateWork("استعلام DB", delaySeconds: 2);

        Console.WriteLine("[Main] أطلقت الثلاث مهام، الآن أنتظرها جميعاً...\n");

        // ننتظر الكل ينتهي — التنفيذ الفعلي قد يكون على thread واحد أو أكثر
        await Task.WhenAll(task1, task2, task3);

        Console.WriteLine("\n[Main] انتهت جميع المهام!");
        // لاحظ: الوقت الكلي ~3 ثوانٍ فقط (وقت أطول مهمة)، وليس 6 ثوانٍ (مجموع الكل)
    }

    static async Task SimulateWork(string taskName, int delaySeconds)
    {
        Console.WriteLine($"  [{taskName}] بدأت... (تأخذ {delaySeconds}ث)");
        await Task.Delay(TimeSpan.FromSeconds(delaySeconds)); // محاكاة انتظار I/O
        Console.WriteLine($"  [{taskName}] ✓ انتهت بعد {delaySeconds}ث");
    }
}

// الناتج المتوقع:
//   [طلب API]    بدأت... (تأخذ 3ث)
//   [قراءة ملف] بدأت... (تأخذ 1ث)
//   [استعلام DB] بدأت... (تأخذ 2ث)
// [Main] أطلقت الثلاث مهام، الآن أنتظرها جميعاً...
//   [قراءة ملف] ✓ انتهت بعد 1ث       ← أول من انتهت
//   [استعلام DB] ✓ انتهت بعد 2ث
//   [طلب API]   ✓ انتهت بعد 3ث
// [Main] انتهت جميع المهام!           ← بعد 3ث فقط وليس 6ث
```

### مثال Parallelism — استغلال فعلي لأنوية المعالج

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class ParallelismExample
{
    static void Main()
    {
        Console.WriteLine("=== Parallelism: توزيع الحسابات على أنوية المعالج ===\n");

        int[] numbers = new int[12]; // 12 عدد لمعالجتها

        // Parallel.For يوزّع الـ iterations على الـ cores المتاحة تلقائياً
        // كل iteration قد يُنفَّذ على core مختلف في نفس اللحظة الزمنية
        Parallel.For(0, numbers.Length, i =>
        {
            int threadId = Thread.CurrentThread.ManagedThreadId;
            numbers[i] = HeavyCalculation(i);
            Console.WriteLine($"  العنصر [{i:D2}] = {numbers[i]:D6}  |  Core/Thread: {threadId}");
        });

        Console.WriteLine($"\nانتهت المعالجة! المجموع = {string.Join(" + ", numbers)[..40]}...");
    }

    // دالة حساب ثقيلة على الـ CPU
    static int HeavyCalculation(int n)
    {
        int result = 0;
        for (int i = 0; i < 1_000_000; i++)
            result += (n + i) % 7;
        return result;
    }
}

// الناتج المتوقع (يتغير كل تشغيل — لاحظ Thread IDs مختلفة تعمل معاً):
//   العنصر [00] = 142857  |  Core/Thread: 4
//   العنصر [01] = 142858  |  Core/Thread: 6   ← thread مختلف في نفس اللحظة!
//   العنصر [02] = 142859  |  Core/Thread: 5
//   العنصر [03] = 142860  |  Core/Thread: 4
//   ...
```

### الفرق الجوهري بين المثالين

ال**Concurrency:**
- ما يحدث فعلاً: تنسيق بين مهام متداخلة زمنياً
- الـ Threads المستخدمة: قد يكون thread واحد فقط (مع async)
- مناسب لـ: I/O — شبكة، ملفات، قاعدة بيانات
- المثال: `Task.WhenAll` لعدة API calls

ال **Parallelism:**
- ما يحدث فعلاً: تنفيذ حقيقي في نفس اللحظة
- الـ Threads المستخدمة: cores/threads متعددة
- مناسب لـ: CPU — حسابات، معالجة صور، ضغط
- المثال: `Parallel.For` لمعالجة بيانات

---

## 4. مكتبة TPL (Task Parallel Library)

هذه المكتبة تجعل الحياة أسهل باستخدام الـ **Tasks** بدلاً من التعامل المباشر مع الـ Threads.

**مثال الكود (استخدام الـ Task):**

```csharp
using System.Threading.Tasks;

class TPL_Example {
    static async Task Main() {
        // إنشاء مهمة (Task) بسهولة
        Task<int> task = Task.Run(() => {
            // كود يأخذ وقتاً طويلاً
            return 42; 
        });

        // القيام بعمل آخر هنا...

        // الحصول على النتيجة عند الجاهزية
        int result = await task;
        Console.WriteLine($"النتيجة هي: {result}");
    }
}
```

### متى نستخدم الـ TPL؟

- ال **`Parallel.ForEach`** — عندما يكون لديك آلاف البيانات وتريد معالجتها بأقصى سرعة (Data Parallelism)
- **`Task.Run`** — عندما تريد تشغيل عملية ثقيلة في الخلفية لكي لا تتعطل شاشة البرنامج (UI)
- **`Task.WhenAll`** — عندما تريد تشغيل 5 طلبات لمواقع مختلفة والانتظار حتى تنتهي جميعها

> **نصيحة مبرمج:** دائماً ابدأ بـ TPL (Tasks) لأنها هي الأحدث والأكثر كفاءة في إدارة موارد الجهاز، ولا تلجأ للـ `Thread` اليدوي إلا في حالات نادرة جداً ومتقدمة.
