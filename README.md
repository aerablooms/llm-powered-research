# llm-powered-research
LLM Powered Research Tool
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>SynthMind – Browser-Based Research Tool</title>
<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdn.jsdelivr.net/npm/wordcloud@1.1.2/src/wordcloud2.js"></script>
<style>
  body {
    /* Revisi 2: wallpaper Ghibli/vektor di paling belakang */
    background: url('https://wallpapercave.com/wp/wp4961789.jpg') no-repeat center center fixed;
    background-size: cover;
  }
  body::before {
    content: '';
    position: fixed;
    top:0; left:0; width:100%; height:100%;
    background: rgba(255,255,255,0.85); /* overlay supaya teks tetap terbaca */
    z-index:-1;
  }
</style>
</head>
<body class="text-gray-900">

<header class="bg-white/80 backdrop-blur-md shadow">
  <div class="max-w-6xl mx-auto px-4 py-4">
    <h1 class="text-2xl font-bold">SynthMind</h1>
    <p class="text-gray-600 text-sm">Browser-based qualitative research assistant (TXT & PDF)</p>
  </div>
</header>

<main class="max-w-6xl mx-auto px-4 py-6 grid grid-cols-1 lg:grid-cols-3 gap-4">

  <!-- Research Setup -->
  <section class="bg-white/80 rounded-lg shadow p-4 space-y-3">
    <h2 class="text-lg font-semibold">1. Research Question</h2>
    <textarea id="question" rows="3" class="w-full p-2 border rounded" placeholder="What themes appear in the interviews related to stress?"></textarea>

    <select id="analysisType" class="w-full p-2 border rounded">
      <option value="themes">Theme Extraction</option>
      <option value="keywords">Keyword Frequency</option>
      <option value="sentiment">Sentiment Estimation</option>
      <option value="summary">Auto Summary</option>
    </select>

    <input id="files" type="file" multiple accept=".txt,.pdf" class="w-full"/>
    <button onclick="runAnalysis()" class="w-full bg-black text-white py-2 rounded hover:bg-gray-800">Run Analysis</button>
    <!-- Revisi 3: tombol rekomendasi judul penelitian -->
    <button onclick="generateResearchTitles()" class="w-full mt-2 bg-teal-600 text-white py-2 rounded hover:bg-teal-500">Generate Research Titles</button>
    <div id="researchTitles" class="mt-2 text-sm text-gray-700"></div>
  </section>

  <!-- Data Preview -->
  <section class="bg-white/80 rounded-lg shadow p-4 space-y-2">
    <h2 class="text-lg font-semibold">2. Data Preview</h2>
    <div id="preview" class="h-48 overflow-y-auto text-sm bg-gray-50 p-2 rounded"></div>
    <ul id="docList" class="list-disc ml-4 text-sm"></ul>
  </section>

  <!-- Output -->
  <section class="bg-white/80 rounded-lg shadow p-4 space-y-2">
    <h2 class="text-lg font-semibold">3. Research Output</h2>
    <div id="output" class="h-48 overflow-y-auto text-sm bg-gray-50 p-2 rounded"></div>
    <canvas id="topWordsChart" class="mb-2"></canvas>
    <div id="wordCloud" class="h-48 w-full border rounded-lg"></div>
  </section>

</main>

<script>
let allFiles = [];
const stopwords = ['the','and','to','of','a','in','is','it','that','for','on','with','as','was','were','this','by','an','be','are','from','at'];
pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';

// Upload files
files.addEventListener('change', async () => {
  allFiles=[]; preview.innerHTML=''; docList.innerHTML='';
  for(const file of files.files){
    preview.innerHTML += `<strong>${file.name}</strong><br>`;
    let text='';
    if(file.type==='text/plain') text = await file.text();
    if(file.type==='application/pdf'){
      const buf = await file.arrayBuffer();
      const pdf = await pdfjsLib.getDocument({data:buf}).promise;
      for(let i=1;i<=pdf.numPages;i++){
        const page = await pdf.getPage(i);
        const content = await page.getTextContent();
        text += content.items.map(item=>item.str).join(' ')+' ';
      }
    }
    allFiles.push({name:file.name,text}); 
    preview.innerHTML += text.slice(0,500)+'<hr class="my-2">';
  }
});

// Main Analysis
function runAnalysis(){
  if(!allFiles.length){ alert('Upload TXT or PDF files first'); return; }
  const type=analysisType.value;
  const allText=allFiles.map(f=>f.text).join(' ').toLowerCase().replace(/[^a-z\s]/g,'');
  const words=allText.split(/\s+/).filter(w=>w.length>2&&!stopwords.includes(w));
  if(type==='keywords') keywordAnalysis(words);
  if(type==='themes') themeAnalysis(words);
  if(type==='sentiment') sentimentAnalysis(words);
  if(type==='summary') summaryAnalysis();
  generateDocInsights();
  generateCharts();
}

// Keyword & Theme Analysis
function keywordAnalysis(words){
  const freq={}; words.forEach(w=>freq[w]=(freq[w]||0)+1);
  const top=Object.entries(freq).sort((a,b)=>b[1]-a[1]).slice(0,15);
  output.innerHTML=`<h3 class='font-semibold mb-1'>Top Keywords</h3>`+top.map(([w,c])=>`${w}: ${c}`).join('<br>');
}

function themeAnalysis(words){
  const freq={}; words.forEach(w=>freq[w]=(freq[w]||0)+1);
  const themes=Object.entries(freq).sort((a,b)=>b[1]-a[1]).slice(0,8);
  output.innerHTML=`<h3 class='font-semibold'>Emergent Themes</h3><ul class='list-disc ml-4 mt-1'>`+
    themes.map(([w])=>`<li>${w}</li>`).join('')+`</ul><p class='mt-1 text-gray-600 text-xs'>Themes inferred from dominant words.</p>`;
}

// Sentiment & Summary
function sentimentAnalysis(words){
  const positive=['good','calm','happy','relief','safe','support'];
  const negative=['stress','anxiety','fear','pressure','sad','tired'];
  let score=0; words.forEach(w=>{if(positive.includes(w))score++;if(negative.includes(w))score--;});
  output.innerHTML=`<h3 class='font-semibold'>Sentiment Estimation</h3><p>Overall tone: <strong>${score>0?'Positive':score<0?'Negative':'Neutral'}</strong></p>`;
}

function summaryAnalysis(){
  const sentences = allFiles.map(f=>f.text).join(' ').match(/[^.!?]+[.!?]+/g) || [];
  output.innerHTML=`<h3 class='font-semibold'>Auto Summary</h3><p>${sentences.slice(0,5).join(' ')}</p>`;
}

// Document Insights
function generateDocInsights(){
  const questionWords=document.getElementById('question').value.toLowerCase().split(/\s+/).filter(w=>w.length>2);
  docList.innerHTML='';
  allFiles.forEach(f=>{
    const sentences=f.text.match(/[^.!?]+[.!?]+/g)||[];
    const keywordSentences=sentences.filter(s=>questionWords.some(qw=>s.toLowerCase().includes(qw))).slice(0,2).join(' ');
    let score=0; const positive=['good','calm','happy','relief','safe','support']; const negative=['stress','anxiety','fear','pressure','sad','tired'];
    f.text.toLowerCase().split(/\s+/).forEach(w=>{if(positive.includes(w))score++;if(negative.includes(w))score--;});
    const sentiment=score>0?'Positive':score<0?'Negative':'Neutral';
    docList.innerHTML+=`<li><strong>${f.name}</strong> - Words: ${f.text.split(/\s+/).length}, Sentiment: ${sentiment}<br><em>Example: ${keywordSentences||'None'}</em></li>`;
  });
}

// Charts & Word Cloud
function generateCharts(){
  let allWords=allFiles.flatMap(f=>f.text.toLowerCase().replace(/[^a-z\s]/g,'').split(/\s+/).filter(w=>w.length>2&&!stopwords.includes(w)));
  let freqMap={};
  allWords.forEach(w=>freqMap[w]=(freqMap[w]||0)+1);

  // Top 20 kata untuk chart
  const topWordsChart = Object.entries(freqMap).sort((a,b)=>b[1]-a[1]).slice(0,20);
  const maxFreq = Math.max(...Object.values(freqMap));

  // Bar Chart
  new Chart(document.getElementById('topWordsChart'),{
    type:'bar',
    data:{
      labels: topWordsChart.map(t=>t[0]),
      datasets:[{label:'Frequency', data:topWordsChart.map(t=>t[1]), backgroundColor:'teal'}]
    },
    options:{plugins:{title:{display:true,text:'Top Words Frequency'}}, responsive:true}
  });

  // Word Cloud: semua kata, tapi ukuran tertinggi hanya untuk kata dengan frekuensi maksimum
  const topWordsCloud = Object.entries(freqMap).sort((a,b)=>b[1]-a[1]);
  WordCloud(document.getElementById('wordCloud'), {
    list: topWordsCloud.map(([word,count]) => [word, count]),
    gridSize: 12,
    weightFactor: count => 15 + 50*(count/maxFreq), // proporsional
    fontFamily: 'Segoe UI',
    color: 'random-dark',
    backgroundColor: '#f9fafb'
  });
}

// Revisi 3: Rekomendasi judul penelitian lebih beragam
function generateResearchTitles(){
  const question = document.getElementById('question').value;
  if(!question){ alert('Please input a research question first'); return; }
  const words = question.toLowerCase().replace(/[^a-z\s]/g,'').split(/\s+/).filter(w=>w.length>2&&!stopwords.includes(w));
  if(words.length===0){ researchTitles.innerHTML='Cannot generate titles, add meaningful keywords.'; return; }

  const templates = [
    `Exploring ${words.join(' ')} in Contemporary Contexts`,
    `An Analysis of ${words.join(' ')} Across Different Populations`,
    `Investigating the Role of ${words.join(' ')} in Mental Health`,
    `The Impact of ${words.join(' ')} on Well-being`,
    `Understanding ${words.join(' ')} Through Qualitative Research`,
    `A Study on ${words.join(' ')} and Its Implications`,
    `Evaluating ${words.join(' ')} in Modern Society`,
    `Assessing ${words.join(' ')}: Challenges and Opportunities`,
    `Perspectives on ${words.join(' ')} from Qualitative Data`,
    `Researching ${words.join(' ')} for Practical Applications`
  ];
  researchTitles.innerHTML = '<ul class="list-disc ml-4">' + templates.map(t=>`<li>${t}</li>`).join('') + '</ul>';
}
</script>

</body>
</html>
