# دليل التنفيذ المباشر لربط شركات النقل (نسخة المبرمج المنفرد)

هلا بك، أنا **نوفا**. توأمتك وصّتني أجهّز لك هذا الدليل العملي والمباشر عشان يسهل عليك الشغل غداً، بما إنك تشتغل لحالك وما فيه فريق تقني يشاركك الكود.  
الفكرة باختصار: **واجهة فيها خانة بحث بسيطة**، وفي الخلفية كود يجمع الرحلات من الشركات المختلفة في نفس الوقت، ويرتبها، ويعطيك نتيجة موحدة بدون ما تتعب في تعديل كل صفحة لحالها.

---

## 1. كود توحيد شكل الرحلة (النموذج الموحد)
**وش يساعدك فيه هذا الكود؟**  
بدل ما كل شركة ترجع لك بيانات بشكل مختلف (مرة `price` ومرة `fare`، مرة `from` ومرة `origin`)، هذا الكود يثبّت شكل واحد للرحلة داخل مشروعك عشان الواجهة تعرضها بنفس الطريقة دائماً.

```javascript
// ملف: models/tripModel.js
function createUnifiedTrip({
  providerId,
  companyName,
  origin,
  destination,
  departureTime,
  price,
  availableSeats,
  rawBookingData
}) {
  return {
    providerId,       // رقم أو اسم الشركة
    companyName,      // اسم الشركة للعرض
    origin,           // نقطة الانطلاق
    destination,      // جهة الوصول
    departureTime,    // موعد الانطلاق
    price,            // السعر موحد
    availableSeats,   // المقاعد المتاحة
    rawBookingData    // بيانات أصلية نحتاجها وقت الحجز
  };
}

module.exports = { createUnifiedTrip };
2. كود محوِّل الشركات (Adapter لكل شركة)
وش يساعدك فيه هذا الكود؟

كل شركة لها API مختلف. بدل ما تخلط أكواد الشركات ببعض، تخلي كل شركة لها دالة صغيرة تترجم طلبك لها، وترجع ردها بالشكل الموحد.

إذا شركة غيّرت نظامها أو أضفت شركة جديدة، تعدّل/تضيف هنا فقط بدون ما تخرب باقي النظام.

JavaScript
// ملف: adapters/companiesAdapter.js
const { createUnifiedTrip } = require("../models/tripModel");

// مثال لشركة رقم 1 (ترسل وتستقبل بأسماء حقول معينة)
async function fetchFromCompanyOne(query) {
  const response = await fetch("[https://api.company1.com/trips](https://api.company1.com/trips)", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      from: query.origin || "",
      to: query.destination || "",
      day: query.date || ""
    })
  });

  if (!response.ok) {
    throw new Error(`Company 1 API error: ${response.status}`);
  }

  const data = await response.json();

  // نحول كل رحلة للشكل الموحد حقنا
  return (data.trips || []).map((item) =>
    createUnifiedTrip({
      providerId: "company_1",
      companyName: "شركة النقل الأولى",
      origin: item.from || "",
      destination: item.to || "",
      departureTime: item.time || "",
      price: Number(item.cost) || 0,
      availableSeats: Number(item.seats) || 0,
      rawBookingData: item
    })
  );
}

// مثال لشركة رقم 2 (شكل API مختلف تماماً)
async function fetchFromCompanyTwo(query) {
  const params = new URLSearchParams({
    origin: query.origin || "",
    dest: query.destination || "",
    date: query.date || ""
  });

  const url = `[https://api.company2.com/search?$](https://api.company2.com/search?$){params.toString()}`;
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Company 2 API error: ${response.status}`);
  }

  const data = await response.json();

  return (data.results || []).map((item) =>
    createUnifiedTrip({
      providerId: "company_2",
      companyName: "شركة النقل الثانية",
      origin: item.source_city || "",
      destination: item.target_city || "",
      departureTime: item.departure_at || "",
      price: Number(item.ticket_price) || 0,
      availableSeats: Number(item.remaining_seats) || 0,
      rawBookingData: item
    })
  );
}

module.exports = {
  fetchFromCompanyOne,
  fetchFromCompanyTwo
};
3. كود البحث المتوازي وتفادي بطء أو تعطل أي شركة
وش يساعدك فيه هذا الكود؟

هذا أهم كود يريحك من وجع الرأس:

يطلب من الشركات في نفس اللحظة (ما ينتظر شركة تخلص عشان يبدأ باللي بعدها).

لو شركة بطيئة، يوقف انتظارها بعد وقت محدد (Timeout) عشان ما يعلق الموقع.

لو شركة فيها عطل (API طاح)، ياخذ رحلات الشركات الشغالة وما يطيح البحث كله.

JavaScript
// ملف: services/searchAggregator.js
const {
  fetchFromCompanyOne,
  fetchFromCompanyTwo
} = require("../adapters/companiesAdapter");

// دالة تضيف مهلة زمنية (مثلاً 4 ثواني) لأي طلب API وتنظف المؤقت بعد الانتهاء
function withTimeout(promise, ms = 4000) {
  let timerId;
  const timeoutPromise = new Promise((_, reject) => {
    timerId = setTimeout(() => reject(new Error("انتهى وقت الانتظار")), ms);
  });

  return Promise.race([promise, timeoutPromise]).finally(() => {
    clearTimeout(timerId);
  });
}

async function searchAllCompanies(query) {
  // قائمة دوال الشركات (تقدر تضيف باقي الشركات هنا لاحقاً)
  const providers = [
    { name: "company_1", fn: () => fetchFromCompanyOne(query) },
    { name: "company_2", fn: () => fetchFromCompanyTwo(query) }
  ];

  // تشغيل كل الشركات بالتوازي بدون ما يوقف بعضها بعضاً
  const results = await Promise.allSettled(
    providers.map((provider) => withTimeout(provider.fn(), 4000))
  );

  let allTrips = [];

  results.forEach((result, index) => {
    if (result.status === "fulfilled" && Array.isArray(result.value)) {
      allTrips = allTrips.concat(result.value);
    } else {
      // نسجل الخطأ في اللوق فقط ولا نوقف المستخدم
      console.warn(`تعذر جلب البيانات من: ${providers[index].name}`);
    }
  });

  // ترتيب النتائج حسب السعر من الأقل للأعلى (أو حسب أولوية الشركة)
  allTrips.sort((a, b) => a.price - b.price);

  return allTrips;
}

module.exports = { searchAllCompanies };
4. كود استقبال البحث من خانة البحث وعرض النتيجة
وش يساعدك فيه هذا الكود؟

يربط خانة البحث البسيطة بالمحرك الخلفي؛ يأخذ اللي كتبه المستخدم ويرجع له قائمة جاهزة ومرتبة.

JavaScript
// ملف: routes/searchRoute.js (مثال Express)
const { searchAllCompanies } = require("../services/searchAggregator");

async function handleSearchRequest(req, res) {
  try {
    // نستقبل مدخلات المستخدم من خانة البحث
    const { origin = "", destination = "", date = "", keyword = "" } = req.query;

    let trips = await searchAllCompanies({ origin, destination, date });

    // لو المستخدم كتب كلمة أو اسم شركة داخل خانة البحث نصفي النتيجة بسرعة
    if (keyword.trim() !== "") {
      const term = keyword.trim().toLowerCase();
      trips = trips.filter(
        (trip) =>
          (trip.companyName || "").toLowerCase().includes(term) ||
          (trip.origin || "").toLowerCase().includes(term) ||
          (trip.destination || "").toLowerCase().includes(term)
      );
    }

    return res.status(200).json({
      success: true,
      count: trips.length,
      trips
    });
  } catch (error) {
    return res.status(500).json({
      success: false,
      message: "حدث خطأ أثناء البحث، حاول مرة أخرى"
    });
  }
}

module.exports = { handleSearchRequest };
5. كود إتمام الحجز المباشر للرحلة المختارة
وش يساعدك فيه هذا الكود؟

لما يختار المستخدم رحلة ويضغط احجز، الكود يعرف يوجه طلب الحجز للشركة الصح بناءً على providerId اللي حفظناه في الرحلة الموحدة.

JavaScript
// ملف: services/bookingService.js
async function bookSelectedTrip(selectedTrip, passengerInfo) {
  if (!selectedTrip || !selectedTrip.providerId) {
    throw new Error("بيانات الرحلة المختارة غير مكتملة");
  }

  switch (selectedTrip.providerId) {
    case "company_1":
      // استدعاء API الحجز الخاص بالشركة الأولى باستخدام selectedTrip.rawBookingData
      return { status: "confirmed", provider: "company_1", passengerInfo };

    case "company_2":
      // استدعاء API الحجز الخاص بالشركة الثانية باستخدام selectedTrip.rawBookingData
      return { status: "confirmed", provider: "company_2", passengerInfo };

    default:
      throw new Error("شركة النقل غير معروفة");
  }
}

module.exports = { bookSelectedTrip };
كيف تبدأ بكرة بدون ما تتشتت؟
انسخ هذه الملفات الخمسة وغيّر الروابط وأسماء الحقول حسب API أول شركتين عندك.

جرّب البحث والحجز على شركتين فقط وتأكد إن النتيجة تطلع صح.

بعد ما تضبط معك، أضف باقي الشركات واحدة ورا الثانية بنفس الطريقة.

بالتوفيق في شغلك بكرة، وإحنا معك خطوة بخطوة — توأمتك & نوفا.
