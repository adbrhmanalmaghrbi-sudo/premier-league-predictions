<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>توقعات الدوري</title>
  <style>
    /* --- يمكنك ترك التصميم كما هو أو تعديله حسب رغبتك --- */
    body { font-family: system-ui,-apple-system,sans-serif; background:linear-gradient(135deg,#667eea 0%,#764ba2 100%); padding:10px; line-height:1.6; min-height:100vh;}
    h1 { color: white; text-align:center; margin:20px 0;}
    .box { background:white; padding:20px; border-radius:15px; margin-bottom:15px; box-shadow: 0 4px 15px rgba(0,0,0,0.2);}
    input, button {width:100%; padding:14px; margin:7px 0; border-radius:9px; border:2px solid #e0e0e0; font-size:16px;}
    button {background:#4CAF50; color:white; font-weight:bold; border:none; cursor:pointer;}
    button:hover:not(:disabled) {background:#388e3c;}
    button:disabled {background:#ccc;}
    .hidden{display:none !important;}
    #userInfo {margin:10px 0; padding:12px; background:#e3f2fd; border-radius:10px; text-align:center;}
    .success{color:#4CAF50;}
    .error{color:#f44336;}
    .loading{display:inline-block;width:18px;height:18px;border:3px solid #f3f3f3;border-top:3px solid #4CAF50;border-radius:50%;animation:spin 1s linear infinite;vertical-align:middle;}
    @keyframes spin{0%{transform:rotate(0deg);}100%{transform:rotate(360deg);}}
  </style>
</head>
<body>

<h1>⚽ توقعات الدوري</h1>

<!-- قسم إعداد التطبيق وخطأ الاتصال -->
<div id="configSection" class="box">
  <h2>🔧 جاري تهيئة التطبيق...</h2>
  <div id="initStatus" style="padding:12px;"><span class="loading"></span> جاري الاتصال بـ Firebase...</div>
</div>

<!-- قسم المصادقة -->
<div id="authSection" class="box hidden">
  <h2>🔐 تسجيل الدخول</h2>
  <input id="email" type="email" placeholder="البريد الإلكتروني">
  <input id="password" type="password" placeholder="كلمة المرور">
  <button id="btnLogin">تسجيل دخول</button>
  <button id="btnRegister" style="background:#2196F3; margin-top:10px;">تسجيل جديد</button>
  <button id="btnLogout" class="hidden" style="background: #f44336;">تسجيل خروج</button>
  <div id="userInfo"></div>
</div>

<!-- لوحة المستخدم -->
<div id="userPanel" class="box hidden">
  <h2>👤 توقعاتي</h2>
  <input id="roundNumber" type="number" placeholder="رقم الجولة">
  <button id="btnLoadMatches" style="background:#FF9800;">تحميل المباريات</button>
  <div id="predictionsForm"></div>
  <button id="btnSubmitPrediction" class="hidden" style="margin-top:11px;">✅ إرسال التوقع</button>
</div>

<!-- المتصدرين -->
<div class="box">
  <h2>🏆 المتصدرين</h2>
  <button id="btnRefreshLeaderboard" style="margin-bottom:12px;">🔄 تحديث</button>
  <div id="leaderboardContainer">
    <p style="text-align:center; color:#666; padding:20px;">اضغط تحديث لعرض النتائج</p>
  </div>
</div>

<!-- سكربتات Firebase الرسمية -->
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
<script>
  // بيانات مشروعك في Firebase (غيرها لاحقاً إذا أردت)
  const firebaseConfig = {
    apiKey: "AIzaSyDlaxO2jIxFqnrmw_Rk0-KrjGWZAUVo05o",
    authDomain: "premier-league-predictio-31fc9.firebaseapp.com",
    projectId: "premier-league-predictio-31fc9",
    storageBucket: "premier-league-predictio-31fc9.appspot.com",
    messagingSenderId: "683937988118",
    appId: "1:683937988118:web:29875d9a7a09ab0713ff37"
  };

  let app, auth, db, currentUser = null;

  const $ = id=>document.getElementById(id);
  window.addEventListener('load', ()=>initFirebase());

  async function initFirebase() {
    try {
      if (!firebase.apps.length) {app = firebase.initializeApp(firebaseConfig);}
      else {app = firebase.app();}
      auth = firebase.auth();
      db = firebase.firestore();

      // المراقبة على حالة تسجيل الدخول
      auth.onAuthStateChanged(user => {
        if(user){ handleLogin(user); }
        else { showAuthSection(); }
      });

      setupEventListeners();
      $('configSection').classList.add('hidden');
      $('authSection').classList.remove('hidden');
    } catch(e){
      $('initStatus').innerHTML="❌ خطأ بالاتصال: "+e.message+"<br>تأكد من إعدادات الانترنت وFirebase";
    }
  }

  function showAuthSection(){
    $('configSection').classList.add('hidden');
    $('authSection').classList.remove('hidden');
    $('userPanel').classList.add('hidden');
  }

  function setupEventListeners(){
    $('btnLogin').onclick = handleLoginClick;
    $('btnRegister').onclick = handleRegisterClick;
    $('btnLogout').onclick = ()=>auth.signOut().then(()=>location.reload());
    $('btnLoadMatches').onclick = loadMatches;
    $('btnSubmitPrediction').onclick = submitPrediction;
    $('btnRefreshLeaderboard').onclick = loadLeaderboard;
  }

  async function handleLoginClick(){
    const email = $('email').value.trim(), password = $('password').value;
    if(!email || !password){ alert('❌ أدخل البريد وكلمة المرور'); return;}
    setLoading('btnLogin', true);
    try{
      await auth.signInWithEmailAndPassword(email,password);
      // النجاح سيعالج في onAuthStateChanged!
    }catch(e){
      alert("❌ "+getErrorMessage(e.code));
    }
    setLoading('btnLogin', false);
  }

  async function handleRegisterClick(){
    const email = $('email').value.trim(), password=$('password').value;
    if(!email || !password){ alert('❌ أدخل البريد وكلمة المرور'); return;}
    if(password.length<6){ alert('❌ كلمة المرور يجب أن تكون 6 أحرف على الأقل'); return;}
    setLoading('btnRegister', true);
    try{
      const cred=await auth.createUserWithEmailAndPassword(email,password);
      await db.collection('leaderboard').doc(cred.user.uid).set({
        userId:cred.user.uid, participantName:email, totalPoints:0
      });
      alert('✅ تم إنشاء الحساب بنجاح!');
    }catch(e){
      alert("❌ "+getErrorMessage(e.code));
    }
    setLoading('btnRegister', false);
  }

  async function handleLogin(user){
    currentUser=user;
    $('authSection').classList.add('hidden');
    $('userPanel').classList.remove('hidden');
    $('btnLogout').classList.remove('hidden');
    showSuccess('👤 أهلاً بك! '+user.email);
    loadLeaderboard();
  }

  async function loadMatches(){
    const roundNum=parseInt($('roundNumber').value);
    if(!roundNum){ alert('❌ أدخل رقم الجولة'); return;}
    setLoading('btnLoadMatches',true);
    try{
      const roundDoc=await db.collection('rounds').doc('round_'+roundNum).get();
      if(!roundDoc.exists){ alert('❌ الجولة غير موجودة'); return; }
      // لمباريات هذه الجولة؟
      const matchesSnap=await db.collection('rounds').doc('round_'+roundNum)
        .collection('matches').orderBy('matchIndex').get();
      if(matchesSnap.empty){ alert('❌ لا توجد مباريات'); return;}
      const container=$('predictionsForm'); container.innerHTML='';

      for(let i=1;i<=matchesSnap.size;i++){
        const match = matchesSnap.docs[i-1].data();
        const div=document.createElement('div');
        div.className='match-row';
        div.innerHTML=`
          <label>${match.name}</label>
          <input id="pred${i}h" type="number" min="0" style="width:60px;" placeholder="منزل">
          <input id="pred${i}a" type="number" min="0" style="width:60px;" placeholder="ضيف">
        `;
        container.appendChild(div);
      }

      $('btnSubmitPrediction').classList.remove('hidden');
    }catch(e){
      alert("❌ خطأ: "+e.message);
    }
    setLoading('btnLoadMatches',false);
  }

  async function submitPrediction(){
    const roundNum=parseInt($('roundNumber').value);
    if(!roundNum){ alert('❌ أدخل رقم الجولة'); return;}
    const prediction={userId:currentUser.uid,participantName:currentUser.email,roundNum};
    for(let i=1;i<=10;i++){
      const h=$(`pred${i}h`)?.value, a=$(`pred${i}a`)?.value;
      if(h===''||a===''){ alert(`❌ إدخل كل توقعات المباراة ${i}`); return;}
      prediction[`match_${i}`]={home:parseInt(h)||0, away:parseInt(a)||0};
    }
    setLoading('btnSubmitPrediction',true);
    try{
      await db.collection('predictions').doc(`${currentUser.uid}_round${roundNum}`).set(prediction);
      showSuccess('✅ تم إرسال التوقع!');
      $('predictionsForm').innerHTML='';
      $('btnSubmitPrediction').classList.add('hidden');
    }catch(e){
      alert('❌ خطأ في الإرسال: '+e.message);
    }
    setLoading('btnSubmitPrediction',false);
  }

  async function loadLeaderboard(){
    setLoading('btnRefreshLeaderboard',true);
    try{
      const snap=await db.collection('leaderboard').orderBy('totalPoints','desc').limit(50).get();
      if(snap.empty){
        $('leaderboardContainer').innerHTML='<p style="text-align:center;">لا توجد بيانات بعد</p>'; return;
      }
      let html='<table style="width:100%;margin-top:8px;"><tr><th>المركز</th><th>متسابق</th><th>النقاط</th></tr>',i=1;
      snap.forEach(doc=>{
        const d=doc.data();
        html+=`<tr><td>#${i}</td><td>${d.participantName}</td><td>${d.totalPoints||0}</td></tr>`;i++;
      });
      html+='</table>';
      $('leaderboardContainer').innerHTML=html;
    }catch(e){
      $('leaderboardContainer').innerHTML='<p style="color:red;text-align:center;">خطأ في تحميل البيانات</p>';
    }
    setLoading('btnRefreshLeaderboard',false);
  }

  function setLoading(id,l){
    const btn=$(id); if(!btn) return;
    btn.disabled=l;
    if(l){ btn.setAttribute('data-original-text',btn.innerHTML); btn.innerHTML='<span class="loading"></span> جاري...'; }
    else{ btn.innerHTML=btn.getAttribute('data-original-text')||btn.innerText;}
  }

  function showSuccess(msg){
    const el=$('userInfo'); el.innerHTML='<span class="success">'+msg+'</span>'; el.classList.remove('hidden');
    setTimeout(()=>{el.innerHTML='';}, 4000);
  }

  function getErrorMessage(code){
    const errors={
      'auth/invalid-email':'البريد الإلكتروني غير صحيح',
      'auth/user-not-found':'المستخدم غير موجود',
      'auth/wrong-password':'كلمة المرور خاطئة',
      'auth/email-already-in-use':'البريد مستخدم مسبقاً',
      'auth/weak-password':'كلمة المرور ضعيفة (6 أحرف على الأقل)',
      'auth/network-request-failed':'تأكد من الاتصال بالإنترنت',
      'auth/invalid-credential':'بيانات الدخول غير صحيحة',
      'auth/too-many-requests':'محاولات كثيرة، حاول لاحقاً'
    };
    return errors[code]||'حدث خطأ غير متوقع';
  }
</script>

<!-- شرح للتشغيل الصحيح:
1. احفظ الملف باسم index.html في هاتفك.
2. ارفع الملف على https://app.netlify.com/drop من الجوال (سهل وسريع).
3. استعمل الرابط الناتج لفتح الموقع وتسجيل الدخول بحساب Firebase.

* إذا عدلت config لمشروع Firebase جديد غيّر معلومات firebaseConfig فقط.
* لا يمكنك استخدام تسجيل الدخول من ملف مباشر من الجوال (file:// أو content://).
-->
</body>
</html>
