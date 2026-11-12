const phaseNames = [
  "address",
  "takeaway",
  "top",
  "downswing",
  "impact",
  "finish",
];

const POINTS = {
  address: document.getElementById("phase-address-score"),
  takeaway: document.getElementById("phase-takeaway-score"),
  top: document.getElementById("phase-top-score"),
  downswing: document.getElementById("phase-downswing-score"),
  impact: document.getElementById("phase-impact-score"),
  finish: document.getElementById("phase-finish-score"),
};

const SUMMARY = {
  address: document.getElementById("phase-address-summary"),
  takeaway: document.getElementById("phase-takeaway-summary"),
  top: document.getElementById("phase-top-summary"),
  downswing: document.getElementById("phase-downswing-summary"),
  impact: document.getElementById("phase-impact-summary"),
  finish: document.getElementById("phase-finish-summary"),
};

const state = {
  pose: null,
  camera: null,
  baseline: null,
  lastLandmarks: null,
  phaseResults: {},
  latestSummary: "",
};

const video = document.getElementById("video");
const overlay = document.getElementById("overlay");
const statusText = document.getElementById("statusText");
const detectedStatus = document.getElementById("detectedStatus");
const currentPhase = document.getElementById("currentPhase");
const overallPoint = document.getElementById("overallPoint");
const saveStatus = document.getElementById("saveStatus");
const captureButton = document.getElementById("captureButton");
const cameraButton = document.getElementById("cameraButton");
const resetButton = document.getElementById("resetButton");
const historyList = document.getElementById("historyList");

const clamp = (value, min, max) => Math.min(Math.max(value, min), max);

function average(values) {
  if (!values.length) return 0;
  return values.reduce((sum, item) => sum + item, 0) / values.length;
}

function point(landmarks, index) {
  const landmark = landmarks[index];
  if (!landmark) return null;
  return { x: landmark.x, y: landmark.y, z: landmark.z || 0 };
}

function midpoint(a, b) {
  return {
    x: (a.x + b.x) / 2,
    y: (a.y + b.y) / 2,
    z: (a.z + b.z) / 2,
  };
}

function distance(a, b) {
  if (!a || !b) return 0;
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function createBaseline(landmarks) {
  const leftShoulder = point(landmarks, 11);
  const rightShoulder = point(landmarks, 12);
  const leftHip = point(landmarks, 23);
  const rightHip = point(landmarks, 24);

  if (!leftShoulder || !rightShoulder || !leftHip || !rightHip) {
    return null;
  }

  return {
    leftShoulder,
    rightShoulder,
    leftHip,
    rightHip,
    shoulderMid: midpoint(leftShoulder, rightShoulder),
    hipMid: midpoint(leftHip, rightHip),
    nose: point(landmarks, 0),
    leftKnee: point(landmarks, 25),
    rightKnee: point(landmarks, 26),
    leftWrist: point(landmarks, 15),
    rightWrist: point(landmarks, 16),
  };
}

function initializePose() {
  if (state.pose) return;

  const pose = new Pose({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`,
  });

  pose.setOptions({
    modelComplexity: 1,
    smoothLandmarks: true,
    minDetectionConfidence: 0.6,
    minTrackingConfidence: 0.6,
    selfieMode: false,
  });

  pose.onResults(handlePoseResults);
  state.pose = pose;
}

async function startCamera() {
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    statusText.textContent = "この端末ではカメラを使用できません。";
    return;
  }

  initializePose();

  try {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: {
        facingMode: "environment",
        width: { ideal: 1280 },
        height: { ideal: 720 },
      },
      audio: false,
    });

    video.srcObject = stream;
    video.onloadedmetadata = () => {
      const canvas = overlay;
      canvas.width = video.videoWidth || 1280;
      canvas.height = video.videoHeight || 720;

      if (!state.camera) {
        state.camera = new Camera(video, {
          onFrame: async () => {
            if (state.pose) {
              await state.pose.send({ image: video });
            }
          },
          width: 1280,
          height: 720,
          facingMode: "environment",
        });
      }

      state.camera.start();
      statusText.textContent = "カメラ起動中。骨格検出を実行します。";
    };
  } catch (error) {
    console.error(error);
    statusText.textContent = "カメラのアクセス許可が必要です。Safari の設定を確認してください。";
  }
}

function handlePoseResults(results) {
  const ctx = overlay.getContext("2d");
  ctx.clearRect(0, 0, overlay.width, overlay.height);

  if (results.image) {
    ctx.drawImage(results.image, 0, 0, overlay.width, overlay.height);
  }

  if (!results.poseLandmarks || !results.poseLandmarks.length) {
    detectedStatus.textContent = "未検出";
    currentPhase.textContent = "待機中";
    return;
  }

  const landmarks = results.poseLandmarks;
  state.lastLandmarks = landmarks;
  detectedStatus.textContent = "検出中";

  drawConnectors(ctx, landmarks, POSE_CONNECTIONS, {
    color: "#49d3b7",
    lineWidth: 3,
  });
  drawLandmarks(ctx, landmarks, {
    color: "#ffffff",
    radius: 4,
  });

  const metrics = evaluatePose(landmarks);
  if (metrics) {
    state.phaseResults = metrics;
    renderPhaseCards(metrics);
    renderOverall(metrics);
  }
}

function evaluatePose(landmarks) {
  const leftShoulder = point(landmarks, 11);
  const rightShoulder = point(landmarks, 12);
  const leftHip = point(landmarks, 23);
  const rightHip = point(landmarks, 24);
  const leftKnee = point(landmarks, 25);
  const rightKnee = point(landmarks, 26);
  const leftWrist = point(landmarks, 15);
  const rightWrist = point(landmarks, 16);
  const nose = point(landmarks, 0);

  if (!leftShoulder || !rightShoulder || !leftHip || !rightHip || !nose) {
    return null;
  }

  const shoulderMid = midpoint(leftShoulder, rightShoulder);
  const hipMid = midpoint(leftHip, rightHip);
  const wristMid = midpoint(leftWrist || leftShoulder, rightWrist || rightShoulder);
  const kneeMid = midpoint(leftKnee || leftHip, rightKnee || rightHip);

  const shoulderHipOffset = Math.abs(shoulderMid.x - hipMid.x);
  const kneeOffset = average([
    Math.abs((leftKnee?.x ?? hipMid.x) - hipMid.x),
    Math.abs((rightKnee?.x ?? hipMid.x) - hipMid.x),
  ]);
  const headOffset = Math.abs(nose.x - shoulderMid.x);
  const wristHeight = Math.abs(wristMid.y - shoulderMid.y);
  const shoulderSpread = Math.abs(leftShoulder.x - rightShoulder.x);
  const hipSpread = Math.abs(leftHip.x - rightHip.x);
  const rotationIndex = Math.max(0, (shoulderSpread + hipSpread) * 0.65);

  const addressScore = clamp(100 - (shoulderHipOffset * 150 + kneeOffset * 140 + headOffset * 180), 0, 100);
  const takeawayScore = clamp((rotationIndex * 260) + (shoulderSpread * 40), 0, 100);
  const topScore = clamp(100 - (Math.abs(wristHeight - 0.2) * 260 + shoulderHipOffset * 200), 0, 100);
  const downswingScore = clamp(100 - (Math.abs(shoulderSpread - hipSpread) * 320 + shoulderHipOffset * 180), 0, 100);
  const impactScore = clamp(100 - (headOffset * 210 + shoulderHipOffset * 240 + Math.abs(wristHeight - 0.18) * 260), 0, 100);
  const finishScore = clamp((rotationIndex * 260 + (1 - Math.min(1, Math.abs(hipMid.x - kneeMid.x) * 2)) * 40), 0, 100);

  const phases = {
    address: {
      score: addressScore,
      summary: addressScore > 75 ? "肩・腰・膝の位置が揃っており、安定したアドレスです。" : addressScore > 50 ? "肩幅と腰の位置を少し整えると良くなります。" : "前傾と体重配分を見直して、肩・腰・膝を揃えましょう。",
    },
    takeaway: {
      score: takeawayScore,
      summary: takeawayScore > 70 ? "腰から肩へ順番よく回転しており、テークバックは良好です。" : takeawayScore > 40 ? "テークバックで肩の回転が少し大きいです。腰と頭の動きを整えましょう。" : "クラブを取りながら腰の回転が弱く、体の動き出しが遅れています。",
    },
    top: {
      score: topScore,
      summary: topScore > 72 ? "トップの位置が整っており、腕と手首の位置が安定しています。" : topScore > 45 ? "トップでの腕の位置が少し崩れています。手首を高く保つ意識を持ってください。" : "トップで腕と体の位置が離れているため、脱力とセットアップを整えると良いです。",
    },
    downswing: {
      score: downswingScore,
      summary: downswingScore > 72 ? "切り返しで腰と肩の順番が良く、下半身から動き出しています。" : downswingScore > 48 ? "腰の動き出しがやや遅く、肩の追随が早い状態です。" : "切り返しの順番を意識して、腰から動き出すようにしましょう。",
    },
    impact: {
      score: impactScore,
      summary: impactScore > 72 ? "インパクト時の姿勢と頭の位置がよく、腕と体のバランスが良好です。" : impactScore > 48 ? "インパクトで頭の位置と体勢が少しぶれています。" : "インパクト時の姿勢が崩れやすいので、体幹と腕のラインを確認しましょう。",
    },
    finish: {
      score: finishScore,
      summary: finishScore > 72 ? "フィニッシュで体が十分に回転し、バランスも安定しています。" : finishScore > 48 ? "フィニッシュの回転量は改善余地があります。" : "フィニッシュで体の軸がぶれやすいので、回転とバランスの意識を強めましょう。",
    },
  };

  return phases;
}

function renderPhaseCards(phaseResults) {
  phaseNames.forEach((phase) => {
    const phaseScore = phaseResults[phase].score;
    const phaseSummary = phaseResults[phase].summary;

    POINTS[phase].textContent = `${Math.round(phaseScore)}点`;
    SUMMARY[phase].textContent = phaseSummary;

    const score = Math.round(phaseScore);
    if (score >= 75) {
      POINTS[phase].style.background = "rgba(73, 211, 183, 0.18)";
      POINTS[phase].style.color = "#7af0d1";
    } else if (score >= 50) {
      POINTS[phase].style.background = "rgba(247, 178, 103, 0.12)";
      POINTS[phase].style.color = "#f7b267";
    } else {
      POINTS[phase].style.background = "rgba(242, 109, 109, 0.12)";
      POINTS[phase].style.color = "#f28a8a";
    }
  });

  const latestPhase = phaseNames
    .map((phase) => ({ phase, score: phaseResults[phase].score }))
    .sort((a, b) => b.score - a.score)[0];

  currentPhase.textContent = latestPhase ? latestPhase.phase : "待機中";
  const summaryText = latestPhase
    ? phaseResults[latestPhase.phase].summary
    : "骨格検出を継続しています。";
  state.latestSummary = summaryText;
}

function renderOverall(results) {
  const allScores = phaseNames.map((phase) => results[phase].score);
  const score = Math.round(average(allScores));
  overallPoint.textContent = `${score}点`;
}

function setBaselineFromCurrent() {
  if (!state.lastLandmarks) {
    statusText.textContent = "骨格が検出されていないため、基準を設定できません。";
    return;
  }

  state.baseline = createBaseline(state.lastLandmarks);
  if (!state.baseline) {
    statusText.textContent = "該当骨格が十分に検出されていません。";
    return;
  }

  statusText.textContent = "基準姿勢を記録しました。診断中です。";
}

async function saveDiagnosis() {
  if (!state.lastLandmarks || !state.phaseResults || !Object.keys(state.phaseResults).length) {
    statusText.textContent = "診断対象の骨格が検出されていません。";
    return;
  }

  const timestamp = new Date();
  const overall = Math.round(
    average(phaseNames.map((phase) => state.phaseResults[phase].score))
  );

  const entry = {
    id: Date.now(),
    createdAt: timestamp.toISOString(),
    overall,
    summary: state.latestSummary,
    snapshot: overlay.toDataURL("image/png"),
    phases: state.phaseResults,
  };

  const dbRequest = indexedDB.open("golfSwingDb", 1);

  dbRequest.onupgradeneeded = () => {
    const db = dbRequest.result;
    if (!db.objectStoreNames.contains("diagnoses")) {
      db.createObjectStore("diagnoses", { keyPath: "id" });
    }
  };

  dbRequest.onsuccess = () => {
    const db = dbRequest.result;
    const tx = db.transaction("diagnoses", "readwrite");
    const store = tx.objectStore("diagnoses");
    store.add(entry);
    tx.oncomplete = () => {
      saveStatus.textContent = "保存済み";
      statusText.textContent = `診断を保存しました（${overall}点）。`;
      renderHistory();
    };
  };

  dbRequest.onerror = () => {
    saveStatus.textContent = "保存失敗";
    statusText.textContent = "診断の保存に失敗しました。";
  };
}

function renderHistory() {
  const request = indexedDB.open("golfSwingDb", 1);

  request.onsuccess = () => {
    const db = request.result;
    const tx = db.transaction("diagnoses", "readonly");
    const store = tx.objectStore("diagnoses");
    const getAll = store.getAll();

    getAll.onsuccess = () => {
      const entries = (getAll.result || []).sort((a, b) => b.id - a.id);
      historyList.innerHTML = "";

      if (!entries.length) {
        historyList.innerHTML = '<li class="empty-state">保存された診断はまだありません。</li>';
        return;
      }

      entries.forEach((entry) => {
        const item = document.createElement("li");
        item.className = "history-item";
        const date = new Date(entry.createdAt).toLocaleString("ja-JP", {
          month: "short",
          day: "numeric",
          hour: "2-digit",
          minute: "2-digit",
        });

        item.innerHTML = `
          <div>
            <strong>${entry.overall}点</strong>
            <div class="history-meta">${date}</div>
          </div>
          <div class="history-meta">${entry.summary || "診断結果"}</div>
        `;
        historyList.appendChild(item);
      });
    };
  };
}

cameraButton.addEventListener("click", startCamera);
captureButton.addEventListener("click", saveDiagnosis);
resetButton.addEventListener("click", setBaselineFromCurrent);

renderHistory();
statusText.textContent = "カメラを起動してください";
currentPhase.textContent = "待機中";

after(() => {
  if (!state.pose) {
    initializePose();
  }
});

function after(callback) {
  requestAnimationFrame(callback);
}
