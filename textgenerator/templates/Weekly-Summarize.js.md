{{#script}} 
// === Text Generator: JavaScript Template (fixed) ===
// 依存：Dataview API（pages/page/date）+ Obsidian Vault API（read）
// 使い方：週次ノートを開く → Text Generator: Run Template → このテンプレを実行

// Dataview API 取得
const dv = app.plugins?.plugins?.dataview?.api || app.plugins.getPlugin?.('dataview')?.api;
if (!dv) {
  return "ERROR: Dataview API が見つかりません（Dataview プラグインを有効化してください）。";
}

// 現在開いているノート（=週次ノート）を取得
const activeFile = app.workspace.getActiveFile();
if (!activeFile) {
  return "ERROR: アクティブなノートが取得できませんでした。週次ノートを前面に表示して再実行してください。";
}

// 週次ノートの Dataview ページ情報（frontmatter など）を取得
const cur = dv.page(activeFile.path);
const start = cur?.week_start ? dv.date(cur.week_start) : null;
const end   = cur?.week_end   ? dv.date(cur.week_end)   : null;

if (!start || !end) {
  return "ERROR: このノートの frontmatter に week_start / week_end (YYYY-MM-DD) を指定してください。";
}

// デイリーノートの格納フォルダ（必要に応じて変更）
const DAILY_FOLDER = '"DailyNote"';

// 対象日のデイリーノート一覧（Dataviewの DataArray）
const pages = dv.pages(DAILY_FOLDER)
  .where(p => p.file?.day && p.file.day >= start && p.file.day <= end)
  .sort(p => p.file.day, 'asc');

// フロントマターを除去
function stripFrontmatter(text) {
  if (!text) return "";
  // 先頭の --- ... --- を最初のクローズまで取り除く（改行差異にも対応）
  return text.replace(/^---\r?\n[\s\S]*?\r?\n---\r?\n?/, "");
}

// Obsidian Vault API で本文を読む（安定）
async function readBody(path) {
  try {
    const file = app.vault.getAbstractFileByPath(path);
    if (!file) return "";
    const raw = await app.vault.read(file);           // ⌛
    return stripFrontmatter(raw);
  } catch {
    return "";
  }
}

// 本文連結
let corpusParts = [];
for (const p of pages) {
  const body = await readBody(p.file.path);
  if (body && body.trim()) corpusParts.push(`# ${p.file.name}\n${body.trim()}`);
}

if (!corpusParts.length) {
  return "現在、週次範囲内のデイリーノート本文が取得できませんでした。\n- フォルダ名（" + DAILY_FOLDER + "）\n- デイリーに file.day が付与されているか\n- week_start / week_end の日付範囲\nを確認してください。";
}

// プロンプト（要約指示 + 入力）
const CORPUS = `=== INPUT START ===
${corpusParts.join("\n\n")}
=== INPUT END ===`;

const INSTRUCTION = `あなたは編集長兼プロジェクトマネージャーです。次の「一週間分のデイリーノート本文」だけに基づき、週次レポートを日本語Markdownで作成してください。
推測や脚色は禁止。出力は次の構造に厳密に従ってください。
日別に重要事項・決定事項・未完了タスク・メトリクス（あれば）を整理して日本語で要約してください。重複は統合し、最後に週次サマリーと来週のTODO も付けてください。
# 出力フォーマット
## 概要（3〜5行）
- 今週の全体像を要約。重要イベント・節目があれば日付込みで記述。

## ハイライト（最大5件）
1. {短い見出し（最大12語）} — {1〜2文の説明}
2. …
   
## 指標 / 実績（任意）
- {KPI/数値が素材にあれば箇条書き。単位・期間・母数を明記}
- {例：PV 12,430（9/1–9/7）, 先週比 +8.2%}

## リスク・課題
- {具体的な阻害要因を1行で/項目ごとに}
- {必要なら根拠となる行やIDも保持}

## 決定事項
- {決まったことを箇条書き、日付・担当・期限があれば付与}

## 次週アクション
- [担当/任意] {具体アクション（締切日）}
- …

## 参考リンク / ノート
- {素材に含まれるファイル名・リンク・IDをそのまま列挙}

# ルール
- 素材外の情報は書かない。確証のない推測・一般論は不可。
- 同内容は統合し、重複を除去。
- 日付・数値・単位は改変しない。
- 文章は簡潔・能動態・箇条書き優先。
`;

return `${INSTRUCTION}\n\n${CORPUS}\n`;

{{/script}}