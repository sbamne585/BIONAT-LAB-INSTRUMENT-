<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1.0"/>
  <title>BioNat Booking w/ Cancel & Email</title>
  <script src="https://cdn.jsdelivr.net/npm/@emailjs/browser@4/dist/email.min.js"></script>
  <style>
    body { font-family:'Segoe UI',sans-serif; background:#eef2f7; margin:0; }
    header { background:#2d3e50; color:#fff; padding:15px 20px; display:flex; justify-content:space-between; align-items:center; }
    header h1 { margin:0; }
    header button { background:#ff7f50; color:#fff; border:none; border-radius:4px; padding:8px 12px; cursor:pointer; }
    .controls { display:flex; gap:15px; padding:15px 20px; }
    select,input,button { font-size:16px; padding:8px; border-radius:4px; border:1px solid #ccc; }
    button.primary { background:#2d3e50; color:#fff; }
    .table-container { margin:20px; overflow-x:auto; background:#fff; border-radius:8px; box-shadow:0 2px 6px rgba(0,0,0,0.1); }
    table { width:100%; border-collapse:collapse; }
    th,td { padding:10px; text-align:center; border:1px solid #ddd; white-space:nowrap; }
    th { background:#f5f7fa; }
    td.slot.available { background:#e8f5e9; color:#2e7d32; cursor:pointer; }
    td.slot.booked    { background:#ffebee; color:#c62828; cursor:pointer; }
    td.slot.past      { background:#f0f0f0; color:#aaa; cursor:not-allowed; }
    .modal-backdrop { position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.4);display:none;justify-content:center;align-items:center; }
    .modal { background:#fff;padding:20px;border-radius:8px;width:320px;box-shadow:0 2px 8px rgba(0,0,0,0.2); }
    .modal h2 { margin:0 0 10px; }
    .modal input { width:100%;padding:8px;margin:6px 0;border:1px solid #ccc;border-radius:4px; }
    .actions { text-align:right; margin-top:10px; }
    .actions button { margin-left:6px;padding:8px 12px;border:none;border-radius:4px;cursor:pointer; }
    .actions .primary { background:#2d3e50; color:#fff; }
    .actions .cancel  { background:#ccc; }
  </style>
</head>
<body>
<header>
  <h1>BioNat Lab Booking</h1>
  <div>
    <button id="adminBtn">Admin Login</button>
    <button id="logoutBtn" style="display:none;">Logout</button>
  </div>
</header>

<div class="controls" id="userControls">
  <label>Instrument: <select id="instrumentSelect"></select></label>
  <label>Date: <input type="date" id="dateInput"/></label>
</div>
<div class="controls" id="adminControls" style="display:none;">
  <button id="addInstBtn" class="primary">Add Instrument</button>
  <button id="editInstBtn" class="primary">Edit Instruments</button>
  <button id="editCredBtn" class="primary">Admin Credentials</button>
</div>

<div class="table-container">
  <table>
    <thead><tr id="headerRow"><th>Time</th></tr></thead>
    <tbody id="bodyContent"></tbody>
  </table>
</div>

<div class="modal-backdrop" id="modalBackdrop"><div class="modal" id="modalContent"></div></div>

<script>
  emailjs.init('YOUR_EMAILJS_PUBLIC_KEY');

  const getData = (k,d) => JSON.parse(localStorage.getItem(k)||JSON.stringify(d));
  const setData = (k,v) => localStorage.setItem(k,JSON.stringify(v));
  if(!localStorage.getItem('instruments')) setData('instruments',['UV-Vis','FTIR','NMR','GC-MS']);
  if(!localStorage.getItem('adminCreds')) setData('adminCreds',{user:'admin',pass:'admin'});
  if(!localStorage.getItem('bookings')) setData('bookings',{});
  let isAdmin=false;

  const backdrop = document.getElementById('modalBackdrop');
  const modal = document.getElementById('modalContent');
  const instrumentSelect = document.getElementById('instrumentSelect');
  const dateInput = document.getElementById('dateInput');
  const headerRow = document.getElementById('headerRow');
  const bodyContent = document.getElementById('bodyContent');

  backdrop.onclick = e => { if(e.target===backdrop) backdrop.style.display='none'; };

  function showModal(html){ modal.innerHTML=html; backdrop.style.display='flex'; }
  function rebuildInst(){ instrumentSelect.innerHTML=''; getData('instruments',[]).forEach(i=>instrumentSelect.add(new Option(i,i))); }
  rebuildInst(); dateInput.value=new Date().toISOString().split('T')[0];

  // Admin: login, add/edit instruments, change credentials
  document.getElementById('adminBtn').onclick = () => showModal(`
    <h2>Admin Login</h2>
    <input id="logUser" placeholder="User"><input id="logPass" type="password" placeholder="Pass">
    <div class="actions"><button class="cancel" onclick="backdrop.click()">Cancel</button><button class="primary" onclick="adminLogin()">Login</button></div>`);
  window.adminLogin = () => {
    const creds=getData('adminCreds',{}),u=modal.querySelector('#logUser').value,p=modal.querySelector('#logPass').value;
    if(u===creds.user&&p===creds.pass){
      isAdmin=true;document.getElementById('userControls').style.display='none';
      document.getElementById('adminControls').style.display='flex';
      document.getElementById('adminBtn').style.display='none';
      document.getElementById('logoutBtn').style.display='inline-block';
      backdrop.style.display='none'; renderCalendar();
    } else alert('Invalid credentials');
  };
  document.getElementById('logoutBtn').onclick = () => {
    isAdmin=false; document.getElementById('userControls').style.display='flex';
    document.getElementById('adminControls').style.display='none';
    document.getElementById('adminBtn').style.display='inline-block';
    document.getElementById('logoutBtn').style.display='none';
    renderCalendar();
  };

  // Instruments management
  document.getElementById('addInstBtn').onclick = () => showModal(`
    <h2>Add Instrument</h2><input id="newInst" placeholder="Name">
    <div class="actions"><button class="cancel" onclick="backdrop.click()">Cancel</button><button class="primary" onclick="addInst()">Add</button></div>`);
  window.addInst = () => {
    const inst=modal.querySelector('#newInst').value.trim(); if(!inst)return alert('Enter name.');
    const arr=getData('instruments',[]);arr.push(inst);setData('instruments',arr);rebuildInst();backdrop.style.display='none';renderCalendar();
  };
  document.getElementById('editInstBtn').onclick = () => {
    const arr=getData('instruments',[]);let html='<h2>Edit Instruments</h2><div>';
    arr.forEach((ins,i)=>html+=`<div><input id="ins${i}" value="${ins}"><button onclick="delInst(${i})">Del</button></div>`);
    html+=`</div><div class="actions"><button class="cancel" onclick="backdrop.click()">Close</button><button class="primary" onclick="saveInsts()">Save</button></div>`;
    showModal(html);
  };
  window.delInst = i => { const arr=getData('instruments',[]);arr.splice(i,1);setData('instruments',arr);rebuildInst();renderCalendar();document.getElementById('editInstBtn').click(); };
  window.saveInsts = () => {
    const arr=Array.from(modal.querySelectorAll('input[id^="ins"]')).map(i=>i.value.trim()).filter(v=>v);
    setData('instruments',arr);rebuildInst();backdrop.style.display='none';renderCalendar();
  };

  // Admin credentials
  document.getElementById('editCredBtn').onclick = () => {
    const c=getData('adminCreds',{}); showModal(`
      <h2>Edit Credentials</h2><input id="edtUser" value="${c.user}"><input id="edtPass" type="password" placeholder="New pass">
      <div class="actions"><button class="cancel" onclick="backdrop.click()">Cancel</button><button class="primary" onclick="saveCred()">Save</button></div>`);
  };
  window.saveCred = () => {
    const u=modal.querySelector('#edtUser').value.trim(), p=modal.querySelector('#edtPass').value.trim();
    if(!u||!p)return alert('Both needed');setData('adminCreds',{user:u,pass:p});backdrop.style.display='none';
  };

  // Booking / cancellation logic
  function bookSlot(inst,dateStr,hr){
    const b=getData('bookings',{});
    b[inst]=b[inst]||{}; b[inst][dateStr]=b[inst][dateStr]||{};
    const ex=b[inst][dateStr][hr];
    if(ex){
      if(confirm(`Cancel booking by ${ex.name}?`)){
        delete b[inst][dateStr][hr];
        emailjs.send('YOUR_SERVICE_ID','YOUR_CANCELLATION_TEMPLATE_ID',{
          to_name:ex.name,to_email:ex.email,
          instrument:inst,date:dateStr,
          time:`${hr.toString().padStart(2,'0')}:00 - ${((hr+1)%24).toString().padStart(2,'0')}:00`
        }).then(()=>alert('Canceled. Email sent.')).catch(()=>alert('Cancel email failed.'));
      }
    } else {
      const name=prompt('Your name:'), email=prompt('Your email:');
      if(!name||!email||!/^\S+@\S+\.\S+$/.test(email))return alert('Valid name & email required.');
      b[inst][dateStr][hr]={name,email};
      emailjs.send('YOUR_SERVICE_ID','YOUR_CONFIRM_TEMPLATE_ID',{
        to_name:name,to_email:email,
        instrument:inst,date:dateStr,
        time:`${hr.toString().padStart(2,'0')}:00 - ${((hr+1)%24).toString().padStart(2,'0')}:00`
      }).then(()=>alert('Booked! Email sent.')).catch(()=>alert('Email failed.'));
    }
    setData('bookings',b); renderCalendar();
  }

  function renderCalendar(){
    const inst=instrumentSelect.value;
    const dt=new Date(dateInput.value);
    const b=getData('bookings',{});
    headerRow.innerHTML='<th>Time</th>'; for(let d=0;d<7;d++){const day=new Date(dt);day.setDate(dt.getDate()+d);headerRow.innerHTML+=`<th>${day.toISOString().split('T')[0]}</th>`;}
    bodyContent.innerHTML='';
    for(let hr=0;hr<24;hr++){
      const tr=document.createElement('tr');
      const from=`${hr.toString().padStart(2,'0')}:00`, to=`${((hr+1)%24).toString().padStart(2,'0')}:00`;
      tr.innerHTML=`<td>${from} – ${to}</td>`;
      for(let d=0;d<7;d++){
        const day=new Date(dt); day.setDate(dt.getDate()+d);
        const dateStr=day.toISOString().split('T')[0];
        const td=document.createElement('td'); td.className='slot';
        if(day < new Date(new Date().toISOString().split('T')[0]+'T00:00')){
          td.classList.add('past'); td.textContent='—';
        } else {
          const ex=b[inst]?.[dateStr]?.[hr];
          if(ex){ td.classList.add('booked'); td.textContent=ex.name; } 
          else { td.classList.add('available'); td.textContent='Available'; }
          td.onclick=()=>bookSlot(inst,dateStr,hr);
        }
        tr.appendChild(td);
      }
      bodyContent.appendChild(tr);
    }
  }

  instrumentSelect.onchange=renderCalendar;
  dateInput.onchange=renderCalendar;
  renderCalendar();
</script>
</body>
</html>
