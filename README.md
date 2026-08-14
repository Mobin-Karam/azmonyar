# azmonyar
کمک دست برای امتحان ها ولی درست استفاده بشه ، همیشه هم در آخر امتحان استفاده بشه دوستون دارم :>

## مرحله اول
- دکمه F12 را بزنید.
- به بخش Console بروید.
- ‍``` allow pasting ``` را در کنسول بنویسیید و اینتر را بزنید.
- کد گفته شده را در کنسول وارد کنید.
- تمام شما یک دستیار برای امتحانات دارید به همین راحتی.
- درست ازش استفاده کبنید ، ستاره هم یادتون نره :)

## مرحله دوم ( کپی کردن اسکریپت به کنسول) :
```js
(() => {
  "use strict";

  // ============================================================
  // آزمونیار Console Edition v3.2
  // مخصوص Moodle / Ravin Academy و سایتهای مشابه
  //
  // FIXES:
  // ✓ تشخیص صحیح شماره سؤال از .rui-qno
  // ✓ استخراج همه سؤالهای موجود در DOM
  // ✓ پشتیبانی از 50+ سؤال در یک صفحه
  // ✓ تشخیص تعداد سؤال از input[name="slots"]
  // ✓ تشخیص سؤالها از quiz navigation
  // ✓ ذخیره سؤالها در localStorage
  // ✓ خروجی TXT شامل Prompt آماده برای ChatGPT
  // ✓ Copy Prompt + Questions
  // ✓ Import answer key
  // ✓ Auto Fill
  // ✓ Onboarding حرفه‌ای ۵ مرحله‌ای
  // ✓ Settings کامل و همگام با Home
  // ✓ Backup JSON + پاک‌سازی امن آزمون فعلی
  // ✓ تنظیم باز شدن پنل و Toast notifications
  // ✓ بدون Submit خودکار
  // ============================================================

  const APP_ID = "__azmoonyar_v32__";
  const STORAGE_KEY = "azmoonyar_v3_state";
  const LETTERS = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";

  // ------------------------------------------------------------
  // If already running, just open panel
  // ------------------------------------------------------------

  if (window.__AZMOONYAR_V32_RUNNING__) {
    document.querySelector("#azy-panel")?.classList.remove("azy-hidden");
    document.querySelector("#azy-bubble")?.classList.add("azy-hidden");
    return;
  }

  window.__AZMOONYAR_V32_RUNNING__ = true;

  // ============================================================
  // STATE
  // ============================================================

  const DEFAULT_STATE = {
    settings: {
      autoCapture: true,
      autoFill: false,
      openOnStart: false,
      notifications: true
    },

    onboarding: {
      completed: false,
      disabled: false,
      remindAfter: 0,
      lastStep: 1
    },

    exams: {}
  };

  function loadState() {
    try {
      const saved = JSON.parse(
        localStorage.getItem(STORAGE_KEY) || "{}"
      );

      return {
        ...DEFAULT_STATE,
        ...saved,

        settings: {
          ...DEFAULT_STATE.settings,
          ...(saved.settings || {})
        },

        onboarding: {
          ...DEFAULT_STATE.onboarding,
          ...(saved.onboarding || {})
        },

        exams: saved.exams || {}
      };
    } catch {
      return JSON.parse(JSON.stringify(DEFAULT_STATE));
    }
  }

  let state = loadState();

  function saveState() {
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify(state)
    );
  }

  // ============================================================
  // HELPERS
  // ============================================================

  const $ = selector => document.querySelector(selector);

  function normalizeText(value = "") {
    return String(value)
      .replace(/\u00a0/g, " ")
      .replace(/\u200e|\u200f/g, "")
      .replace(/[ \t]+/g, " ")
      .replace(/\n[ \t]+/g, "\n")
      .trim();
  }

  function toEnglishDigits(value = "") {
    const fa = "۰۱۲۳۴۵۶۷۸۹";
    const ar = "٠١٢٣٤٥٦٧٨٩";

    return String(value)
      .replace(/[۰-۹]/g, d => String(fa.indexOf(d)))
      .replace(/[٠-٩]/g, d => String(ar.indexOf(d)));
  }

  function escapeHtml(value = "") {
    return String(value).replace(
      /[&<>"']/g,
      c => ({
        "&": "&amp;",
        "<": "&lt;",
        ">": "&gt;",
        '"': "&quot;",
        "'": "&#039;"
      })[c]
    );
  }

  function formatBytes(bytes = 0) {
    const value = Number(bytes) || 0;

    if (value < 1024) {
      return `${value} B`;
    }

    if (value < 1024 * 1024) {
      return `${(value / 1024).toFixed(1)} KB`;
    }

    return `${(value / (1024 * 1024)).toFixed(1)} MB`;
  }

  function getStorageSize() {
    try {
      return new Blob([
        localStorage.getItem(STORAGE_KEY) || ""
      ]).size;
    } catch {
      return 0;
    }
  }

  function getSavedAnswerCount() {
    return Object.keys(
      getExam().answers || {}
    ).length;
  }

  // ============================================================
  // EXAM ID
  // ============================================================

  function getExamKey() {
    const url = new URL(location.href);

    const cmid =
      url.searchParams.get("cmid") ||
      $("input[name='cmid']")?.value ||
      "unknown";

    const attempt =
      url.searchParams.get("attempt") ||
      $("input[name='attempt']")?.value ||
      "default";

    return `${location.hostname}:${cmid}:${attempt}`;
  }

  function getExamTitle() {
    return normalizeText(
      $("h1")?.textContent ||
      $(".page-header-headings")?.textContent ||
      document.title ||
      "آزمون"
    );
  }

  function getExam() {
    const key = getExamKey();

    if (!state.exams[key]) {
      state.exams[key] = {
        title: getExamTitle(),
        createdAt: Date.now(),
        updatedAt: Date.now(),
        questions: {},
        answers: {}
      };
    }

    state.exams[key].title = getExamTitle();

    return state.exams[key];
  }

  // ============================================================
  // EXPECTED QUESTION COUNT
  // ============================================================

  function getExpectedQuestionCount() {
    // Moodle puts all slots here:
    // value="1,2,3,...,50"

    const slots = $("input[name='slots']");

    if (slots?.value) {
      const list = slots.value
        .split(",")
        .map(x => Number(x.trim()))
        .filter(Number.isFinite);

      if (list.length) {
        return list.length;
      }
    }

    // Fallback: quiz navigation buttons

    const buttons = document.querySelectorAll(
      ".qn_buttons .qnbutton, [id^='quiznavbutton']"
    );

    if (buttons.length) {
      return buttons.length;
    }

    return getQuestionContainers().length;
  }

  // ============================================================
  // QUESTION CONTAINERS
  // ============================================================

  function getQuestionContainers() {
    let questions = [
      ...document.querySelectorAll(".que")
    ];

    if (!questions.length) {
      questions = [
        ...document.querySelectorAll(
          [
            "[id^='question-']",
            "[data-region='question']",
            ".question"
          ].join(",")
        )
      ];
    }

    return questions.filter(
      (node, index, all) =>
        !all.some(
          (other, otherIndex) =>
            index !== otherIndex &&
            other.contains(node)
        )
    );
  }

  // ============================================================
  // FIXED QUESTION NUMBER DETECTION
  // ============================================================

  function getQuestionNumber(container, fallbackIndex) {

    // BEST source for your Moodle:
    //
    // <span class="rui-qno">1</span>

    const ruiNumber = container.querySelector(
      ".rui-qno"
    );

    if (ruiNumber) {
      const n = Number(
        toEnglishDigits(
          ruiNumber.textContent
        ).match(/\d+/)?.[0]
      );

      if (Number.isFinite(n)) {
        return n;
      }
    }

    // Other Moodle themes

    const qno = container.querySelector(
      ".qno"
    );

    if (qno) {
      const n = Number(
        toEnglishDigits(
          qno.textContent
        ).match(/\d+/)?.[0]
      );

      if (Number.isFinite(n)) {
        return n;
      }
    }

    // Input name:
    //
    // q2677:1_answer
    // q2677:25_answer

    const answerInput =
      container.querySelector(
        [
          "input[type='radio'][name]",
          "input[type='checkbox'][name]",
          "select[name]"
        ].join(",")
      );

    if (answerInput?.name) {
      const match = toEnglishDigits(
        answerInput.name
      ).match(
        /^q\d+:(\d+)_/
      );

      if (match) {
        return Number(match[1]);
      }
    }

    // sequencecheck is also reliable

    const sequenceInput =
      container.querySelector(
        "input[name*='sequencecheck']"
      );

    if (sequenceInput?.name) {
      const match = toEnglishDigits(
        sequenceInput.name
      ).match(
        /^q\d+:(\d+)_/
      );

      if (match) {
        return Number(match[1]);
      }
    }

    // ID:
    //
    // question-2677-1
    //
    // IMPORTANT:
    // use LAST number, NOT first number.

    if (container.id) {
      const match = toEnglishDigits(
        container.id
      ).match(
        /-(\d+)$/
      );

      if (match) {
        return Number(match[1]);
      }
    }

    return fallbackIndex + 1;
  }

  // ============================================================
  // OPTION EXTRACTION
  // ============================================================

  function getOptionInputs(container) {
    let inputs = [
      ...container.querySelectorAll(
        ".answer input[type='radio']:not([value='-1']), .answer input[type='checkbox']"
      )
    ];

    if (!inputs.length) {
      inputs = [
        ...container.querySelectorAll(
          "input[type='radio']:not([value='-1']), input[type='checkbox']"
        )
      ];
    }

    // Ignore Moodle flag inputs, etc.

    return inputs.filter(input =>
      /_answer/.test(input.name || "")
    );
  }

  function getOptionText(input, container) {

    // Ravin/Moodle:
    //
    // aria-labelledby="q2677:1_answer0_label"

    const labelledBy =
      input.getAttribute(
        "aria-labelledby"
      );

    if (labelledBy) {
      const labelElement =
        document.getElementById(
          labelledBy
        );

      if (labelElement) {
        const clone =
          labelElement.cloneNode(true);

        // Remove "A." because we add it ourselves

        clone
          .querySelectorAll(".answernumber")
          .forEach(x => x.remove());

        return normalizeText(
          clone.textContent
        );
      }
    }

    // label[for=id]

    if (input.id) {
      try {
        const label =
          container.querySelector(
            `label[for="${CSS.escape(input.id)}"]`
          );

        if (label) {
          return normalizeText(
            label.textContent
          );
        }
      } catch {}
    }

    // Closest option row

    const row = input.closest(
      ".r0, .r1, .answer > div, .form-check, li"
    );

    if (row) {
      const clone =
        row.cloneNode(true);

      clone
        .querySelectorAll(
          "input,button,.answernumber"
        )
        .forEach(x => x.remove());

      return normalizeText(
        clone.textContent
      );
    }

    return "";
  }

  // ============================================================
  // EXTRACT QUESTION
  // ============================================================

  function extractQuestion(container, index) {
    const number =
      getQuestionNumber(
        container,
        index
      );

    let questionText = "";

    const qtext =
      container.querySelector(
        ".qtext"
      );

    if (qtext) {
      questionText =
        normalizeText(
          qtext.textContent
        );
    }

    // fallback

    if (!questionText) {
      const content =
        container.querySelector(
          ".formulation"
        );

      if (content) {
        const clone =
          content.cloneNode(true);

        clone
          .querySelectorAll(
            [
              ".answer",
              ".ablock",
              ".prompt",
              ".accesshide",
              "input",
              "button",
              "select"
            ].join(",")
          )
          .forEach(x => x.remove());

        questionText =
          normalizeText(
            clone.textContent
          );
      }
    }

    const inputs =
      getOptionInputs(container);

    const options =
      inputs.map(
        (input, optionIndex) => ({
          letter:
            LETTERS[optionIndex] ||
            String(optionIndex + 1),

          text:
            getOptionText(
              input,
              container
            )
        })
      );

    return {
      number,
      text: questionText,
      options,
      updatedAt: Date.now()
    };
  }

  // ============================================================
  // SCAN ALL QUESTIONS
  // ============================================================

  function scanAllQuestions(
    silent = false
  ) {
    const exam = getExam();

    const containers =
      getQuestionContainers();

    let changed = 0;

    containers.forEach(
      (container, index) => {

        const question =
          extractQuestion(
            container,
            index
          );

        if (
          !question.text &&
          !question.options.length
        ) {
          return;
        }

        const key =
          String(
            question.number
          );

        const previous =
          exam.questions[key];

        exam.questions[key] = {
          ...(previous || {}),

          ...question,

          firstSeenAt:
            previous?.firstSeenAt ||
            Date.now()
        };

        if (
          JSON.stringify(previous) !==
          JSON.stringify(
            exam.questions[key]
          )
        ) {
          changed++;
        }
      }
    );

    exam.updatedAt =
      Date.now();

    saveState();

    updateUI();

    if (
      state.settings.autoFill
    ) {
      fillCurrentAnswers(true);
    }

    if (!silent) {
      const expected =
        getExpectedQuestionCount();

      toast(
        `${Object.keys(exam.questions).length}/${expected} سؤال ذخیره شد`
      );
    }

    return changed;
  }

  // ============================================================
  // SORT QUESTIONS
  // ============================================================

  function getSortedQuestions() {
    return Object.values(
      getExam().questions
    ).sort(
      (a, b) =>
        Number(a.number) -
        Number(b.number)
    );
  }

  // ============================================================
  // PROMPT FOR CHATGPT
  // ============================================================

  function getChatGPTPrompt() {
    const count =
      getSortedQuestions().length;

    return `
I have a practice exam with ${count} multiple-choice questions.

Please answer ALL questions in this file.

IMPORTANT OUTPUT RULES:

1. Return ONLY the answer key.
2. Do NOT explain the answers.
3. Do NOT repeat the questions.
4. Use exactly this format:

1. D
2. B
3. A
4. C

5. Keep the same question numbers.
6. Answer every question.
7. Each answer must be only one letter: A, B, C, D, etc.
8. Put exactly one answer per line.
9. Do not use Markdown tables.
10. Do not include any introduction or conclusion.

The result will be pasted directly into my practice-exam helper,
so the formatting must remain exactly:

QUESTION_NUMBER. LETTER

Example:

1. D
2. B
3. A
4. C

Now answer the questions below:
`.trim();
  }

  // ============================================================
  // BUILD TXT
  // ============================================================

  function buildOutputText() {
    // Make sure latest DOM state is captured first.

    scanAllQuestions(true);

    const exam =
      getExam();

    const questions =
      getSortedQuestions();

    const expected =
      getExpectedQuestionCount();

    const lines = [];

    // ----------------------------------------------------------
    // PROMPT FIRST
    // ----------------------------------------------------------

    lines.push(
      "============================================================"
    );

    lines.push(
      "PROMPT FOR CHATGPT"
    );

    lines.push(
      "============================================================"
    );

    lines.push("");

    lines.push(
      getChatGPTPrompt()
    );

    lines.push("");
    lines.push("");

    // ----------------------------------------------------------
    // INFO
    // ----------------------------------------------------------

    lines.push(
      "============================================================"
    );

    lines.push(
      "EXAM INFORMATION"
    );

    lines.push(
      "============================================================"
    );

    lines.push("");

    lines.push(
      `Title: ${exam.title}`
    );

    lines.push(
      `Detected questions: ${questions.length}`
    );

    lines.push(
      `Expected questions: ${expected}`
    );

    lines.push("");

    // Don't expose session tokens; clean URL

    const safeUrl =
      new URL(location.href);

    safeUrl.searchParams.delete(
      "sesskey"
    );

    lines.push(
      `Source: ${safeUrl.toString()}`
    );

    lines.push("");

    lines.push(
      "============================================================"
    );

    lines.push(
      "QUESTIONS"
    );

    lines.push(
      "============================================================"
    );

    lines.push("");

    // ----------------------------------------------------------
    // QUESTIONS
    // ----------------------------------------------------------

    questions.forEach(
      question => {

        lines.push(
          `QUESTION ${question.number}`
        );

        lines.push("");

        lines.push(
          question.text ||
          "[Question text not detected]"
        );

        lines.push("");

        question.options.forEach(
          option => {
            lines.push(
              `${option.letter}. ${option.text}`
            );
          }
        );

        lines.push("");

        lines.push(
          "------------------------------------------------------------"
        );

        lines.push("");
      }
    );

    // ----------------------------------------------------------
    // ANSWER FORMAT REMINDER
    // ----------------------------------------------------------

    lines.push(
      "============================================================"
    );

    lines.push(
      "REQUIRED ANSWER FORMAT"
    );

    lines.push(
      "============================================================"
    );

    lines.push("");

    lines.push(
      "Return ONLY:"
    );

    lines.push("");

    lines.push(
      "1. A"
    );

    lines.push(
      "2. B"
    );

    lines.push(
      "3. C"
    );

    lines.push(
      "..."
    );

    lines.push("");

    lines.push(
      `There should be exactly ${questions.length} answer lines.`
    );

    return lines.join("\n");
  }

  // ============================================================
  // COPY
  // ============================================================

  async function copyOutput() {
    const text =
      buildOutputText();

    try {
      await navigator.clipboard.writeText(
        text
      );

      toast(
        `Prompt + ${getSortedQuestions().length} سؤال کپی شد`
      );
    } catch {

      const textarea =
        document.createElement(
          "textarea"
        );

      textarea.value =
        text;

      textarea.style.cssText =
        `
          position:fixed;
          left:-999999px;
          top:-999999px;
        `;

      document.body.appendChild(
        textarea
      );

      textarea.select();

      document.execCommand(
        "copy"
      );

      textarea.remove();

      toast(
        `Prompt + ${getSortedQuestions().length} سؤال کپی شد`
      );
    }
  }

  // ============================================================
  // DOWNLOAD TXT
  // ============================================================

  function downloadOutput() {
    const text =
      buildOutputText();

    const blob =
      new Blob(
        [text],
        {
          type:
            "text/plain;charset=utf-8"
        }
      );

    const url =
      URL.createObjectURL(
        blob
      );

    const anchor =
      document.createElement(
        "a"
      );

    const title =
      getExamTitle()
        .replace(
          /[\\/:*?"<>|]+/g,
          "-"
        )
        .slice(0, 60);

    anchor.href =
      url;

    anchor.download =
      `azmoonyar-${title || "practice-exam"}-${getSortedQuestions().length}-questions.txt`;

    document.body.appendChild(
      anchor
    );

    anchor.click();

    anchor.remove();

    setTimeout(
      () =>
        URL.revokeObjectURL(
          url
        ),
      2000
    );

    toast(
      `TXT با ${getSortedQuestions().length} سؤال ساخته شد`
    );
  }

  // ============================================================
  // SETTINGS / BACKUP
  // ============================================================

  function downloadBackup() {
    const backup = {
      app: "AzmoonYar Console",
      version: "3.2.0",
      exportedAt: new Date().toISOString(),
      hostname: location.hostname,
      state
    };

    const blob = new Blob(
      [JSON.stringify(backup, null, 2)],
      {
        type: "application/json;charset=utf-8"
      }
    );

    const url =
      URL.createObjectURL(
        blob
      );

    const anchor =
      document.createElement(
        "a"
      );

    anchor.href =
      url;

    anchor.download =
      `azmoonyar-backup-${new Date().toISOString().slice(0, 10)}.json`;

    document.body.appendChild(
      anchor
    );

    anchor.click();
    anchor.remove();

    setTimeout(
      () =>
        URL.revokeObjectURL(
          url
        ),
      2000
    );

    toast(
      "پشتیبان JSON آزمونیار دانلود شد.",
      "success"
    );
  }

  function clearCurrentExamData() {
    const key =
      getExamKey();

    delete state.exams[key];

    saveState();

    getExam();

    const answerArea =
      $("#azy-answer-key");

    if (answerArea) {
      answerArea.value = "";
    }

    const results =
      $("#azy-results");

    if (results) {
      results.innerHTML = "";
    }

    updateUI();

    toast(
      "دادههای ذخیرهشده همین آزمون پاک شدند.",
      "success",
      3200,
      true
    );
  }

  // ============================================================
  // ANSWER KEY PARSER
  // ============================================================

  function parseAnswerKey(raw) {
    raw =
      toEnglishDigits(
        String(raw || "")
      ).trim();

    if (!raw) {
      return {};
    }

    // ----------------------------------------------------------
    // JSON
    // ----------------------------------------------------------

    try {
      const parsed =
        JSON.parse(raw);

      const result = {};

      if (
        Array.isArray(parsed)
      ) {
        parsed.forEach(
          (value, index) => {
            const letter =
              String(value)
                .toUpperCase()
                .match(/[A-Z]/)?.[0];

            if (letter) {
              result[
                String(index + 1)
              ] = letter;
            }
          }
        );
      }

      if (
        parsed &&
        typeof parsed === "object" &&
        !Array.isArray(parsed)
      ) {
        Object.entries(parsed)
          .forEach(
            ([number, value]) => {

              const n =
                String(number)
                  .match(/\d+/)?.[0];

              const letter =
                String(value)
                  .toUpperCase()
                  .match(/[A-Z]/)?.[0];

              if (n && letter) {
                result[
                  String(Number(n))
                ] = letter;
              }
            }
          );
      }

      if (
        Object.keys(result).length
      ) {
        return result;
      }
    } catch {}

    // ----------------------------------------------------------
    // Numbered:
    //
    // 1. D
    // 2. B
    // ----------------------------------------------------------

    const numbered = {};

    raw
      .split(/\r?\n/)
      .forEach(line => {

        const match =
          line.match(
            /^\s*(\d+)\s*[\.\-:)=،]\s*[\(\[]?\s*([A-Z])\b/i
          );

        if (match) {
          numbered[
            String(Number(match[1]))
          ] =
            match[2].toUpperCase();
        }
      });

    if (
      Object.keys(numbered).length
    ) {
      return numbered;
    }

    // ----------------------------------------------------------
    // Compact:
    //
    // DBACABCD...
    // ----------------------------------------------------------

    const compact =
      raw
        .toUpperCase()
        .replace(
          /[^A-Z]/g,
          ""
        );

    if (compact) {
      const result = {};

      [...compact].forEach(
        (letter, index) => {
          result[
            String(index + 1)
          ] = letter;
        }
      );

      return result;
    }

    return {};
  }

  // ============================================================
  // SAVE ANSWERS
  // ============================================================

  function saveAnswerKey() {
    const raw =
      $("#azy-answer-key")?.value ||
      "";

    const answers =
      parseAnswerKey(raw);

    const count =
      Object.keys(
        answers
      ).length;

    if (!count) {
      toast(
        "هیچ پاسخ معتبری پیدا نشد"
      );

      return;
    }

    getExam().answers =
      answers;

    saveState();

    updateUI();

    toast(
      `${count} پاسخ ذخیره شد`
    );
  }

  // ============================================================
  // FILL
  // ============================================================

  function selectAnswer(
    container,
    letter
  ) {
    const index =
      LETTERS.indexOf(
        String(letter).toUpperCase()
      );

    if (index < 0) {
      return false;
    }

    const inputs =
      getOptionInputs(
        container
      );

    const input =
      inputs[index];

    if (!input) {
      return false;
    }

    if (!input.checked) {

      input.click();

      input.dispatchEvent(
        new Event(
          "input",
          {
            bubbles: true
          }
        )
      );

      input.dispatchEvent(
        new Event(
          "change",
          {
            bubbles: true
          }
        )
      );
    }

    return true;
  }

  function fillCurrentAnswers(
    silent = false
  ) {
    const answers =
      getExam().answers;

    let filled = 0;
    let missing = 0;

    getQuestionContainers()
      .forEach(
        (container, index) => {

          const number =
            getQuestionNumber(
              container,
              index
            );

          const answer =
            answers[
              String(number)
            ];

          if (!answer) {
            missing++;
            return;
          }

          if (
            selectAnswer(
              container,
              answer
            )
          ) {
            filled++;
          }
        }
      );

    if (!silent) {
      toast(
        `${filled} پاسخ انتخاب شد${missing ? ` • ${missing} بدون کلید` : ""}`
      );
    }
  }

  // ============================================================
  // CHECK
  // ============================================================

  function getSelectedLetter(
    container
  ) {
    const inputs =
      getOptionInputs(
        container
      );

    const index =
      inputs.findIndex(
        input =>
          input.checked
      );

    return index >= 0
      ? LETTERS[index]
      : null;
  }

  function checkAnswers() {
    const answers =
      getExam().answers;

    const results = [];

    let correct = 0;
    let wrong = 0;
    let empty = 0;
    let noKey = 0;

    getQuestionContainers()
      .forEach(
        (container, index) => {

          const number =
            getQuestionNumber(
              container,
              index
            );

          const expected =
            answers[
              String(number)
            ];

          const selected =
            getSelectedLetter(
              container
            );

          let status;

          if (!expected) {
            status = "nokey";
            noKey++;
          } else if (!selected) {
            status = "empty";
            empty++;
          } else if (
            selected === expected
          ) {
            status =
              "correct";

            correct++;
          } else {
            status =
              "wrong";

            wrong++;
          }

          results.push({
            number,
            selected,
            expected,
            status
          });
        }
      );

    const box =
      $("#azy-results");

    box.innerHTML =
      results
        .map(result => {

          let text;

          switch (
            result.status
          ) {

            case "correct":
              text =
                `✓ ${result.selected}`;
              break;

            case "wrong":
              text =
                `${result.selected} → ${result.expected}`;
              break;

            case "empty":
              text =
                `— → ${result.expected}`;
              break;

            default:
              text =
                "بدون کلید";
          }

          return `
            <div class="azy-result ${result.status}">
              <b>${result.number}</b>
              <span>${text}</span>
            </div>
          `;
        })
        .join("");

    switchTab(
      "answers"
    );

    toast(
      `✓ ${correct} • ✗ ${wrong} • خالی ${empty} • بدون کلید ${noKey}`
    );
  }

  // ============================================================
  // UI STYLES
  // ============================================================

  const style =
    document.createElement(
      "style"
    );

  style.textContent = `
    #${APP_ID}{
      --azy-bg:#08111f;
      --azy-surface:#0d1728;
      --azy-surface-2:#111e31;
      --azy-surface-3:#17243a;
      --azy-border:rgba(148,163,184,.16);
      --azy-border-strong:rgba(148,163,184,.28);
      --azy-text:#f8fafc;
      --azy-muted:#94a3b8;
      --azy-primary:#6366f1;
      --azy-primary-2:#8b5cf6;
      --azy-success:#10b981;
      --azy-warning:#f59e0b;
      --azy-danger:#ef4444;
      --azy-info:#38bdf8;
      --azy-shadow:0 30px 90px rgba(2,6,23,.58);
      --azy-radius:22px;

      all:initial;
      direction:rtl;
      font-family:Tahoma,Arial,sans-serif;
      color:var(--azy-text);
      z-index:2147483647;
    }

    #${APP_ID} *,
    #${APP_ID} *::before,
    #${APP_ID} *::after{
      box-sizing:border-box;
      font-family:inherit;
    }

    #${APP_ID} button,
    #${APP_ID} textarea,
    #${APP_ID} input{
      font:inherit;
    }

    #${APP_ID} button:focus-visible,
    #${APP_ID} textarea:focus-visible,
    #${APP_ID} input:focus-visible{
      outline:2px solid rgba(129,140,248,.95);
      outline-offset:2px;
    }

    /* ---------------------------------------------------------
       Floating bubble
    --------------------------------------------------------- */

    #azy-bubble{
      position:fixed;
      right:22px;
      bottom:22px;
      width:64px;
      height:64px;
      padding:0;
      border:1px solid rgba(255,255,255,.2);
      border-radius:50%;
      cursor:pointer;
      isolation:isolate;
      display:grid;
      place-items:center;
      background:
        radial-gradient(circle at 30% 25%,rgba(255,255,255,.28),transparent 27%),
        linear-gradient(145deg,var(--azy-primary),var(--azy-primary-2));
      color:#fff;
      font-size:25px;
      font-weight:900;
      box-shadow:
        0 16px 38px rgba(79,70,229,.38),
        0 8px 22px rgba(2,6,23,.38);
      transition:
        transform .2s ease,
        box-shadow .2s ease,
        filter .2s ease;
      z-index:2147483647;
    }

    #azy-bubble::before{
      content:"";
      position:absolute;
      inset:-7px;
      border-radius:inherit;
      border:1px solid rgba(129,140,248,.25);
      animation:azyPulse 2.3s ease-out infinite;
      z-index:-1;
    }

    #azy-bubble:hover{
      transform:translateY(-3px) scale(1.035);
      filter:saturate(1.08);
      box-shadow:
        0 20px 46px rgba(79,70,229,.48),
        0 10px 28px rgba(2,6,23,.42);
    }

    #azy-bubble:active{
      transform:scale(.97);
    }

    #azy-bubble.azy-hidden{
      display:none;
    }

    #azy-badge{
      position:absolute;
      top:-5px;
      left:-6px;
      min-width:25px;
      height:25px;
      padding:0 6px;
      display:grid;
      place-items:center;
      border-radius:999px;
      background:linear-gradient(145deg,#059669,var(--azy-success));
      border:2px solid #08111f;
      color:#fff;
      font-size:10px;
      font-weight:800;
      box-shadow:0 4px 10px rgba(16,185,129,.28);
    }

    @keyframes azyPulse{
      0%{opacity:.8;transform:scale(.9)}
      75%,100%{opacity:0;transform:scale(1.22)}
    }

    /* ---------------------------------------------------------
       Main panel
    --------------------------------------------------------- */

    #azy-panel{
      position:fixed;
      right:18px;
      bottom:18px;
      width:min(448px,calc(100vw - 24px));
      max-height:90vh;
      overflow:hidden;
      border:1px solid var(--azy-border-strong);
      border-radius:var(--azy-radius);
      background:
        radial-gradient(circle at 100% 0%,rgba(99,102,241,.13),transparent 30%),
        linear-gradient(180deg,rgba(13,23,40,.985),rgba(8,17,31,.985));
      box-shadow:var(--azy-shadow);
      backdrop-filter:blur(22px) saturate(1.15);
      -webkit-backdrop-filter:blur(22px) saturate(1.15);
      z-index:2147483647;
      animation:azyPanelIn .22s cubic-bezier(.2,.8,.2,1);
    }

    #azy-panel.azy-hidden{
      display:none;
    }

    @keyframes azyPanelIn{
      from{opacity:0;transform:translateY(14px) scale(.975)}
      to{opacity:1;transform:translateY(0) scale(1)}
    }

    .azy-header{
      position:relative;
      display:flex;
      justify-content:space-between;
      align-items:center;
      gap:12px;
      min-height:70px;
      padding:13px 14px;
      background:
        linear-gradient(180deg,rgba(255,255,255,.035),rgba(255,255,255,0));
      border-bottom:1px solid var(--azy-border);
    }

    .azy-header::after{
      content:"";
      position:absolute;
      right:14px;
      left:14px;
      bottom:-1px;
      height:1px;
      background:linear-gradient(90deg,transparent,rgba(99,102,241,.55),transparent);
      pointer-events:none;
    }

    .azy-brand{
      min-width:0;
      display:flex;
      align-items:center;
      gap:11px;
    }

    .azy-logo{
      width:42px;
      height:42px;
      flex:0 0 42px;
      display:grid;
      place-items:center;
      border:1px solid rgba(255,255,255,.15);
      border-radius:14px;
      background:
        radial-gradient(circle at 25% 20%,rgba(255,255,255,.28),transparent 30%),
        linear-gradient(145deg,var(--azy-primary),var(--azy-primary-2));
      box-shadow:0 8px 22px rgba(79,70,229,.3);
      color:#fff;
      font-size:21px;
      font-weight:900;
    }

    .azy-brand-copy{
      min-width:0;
    }

    .azy-brand-title-row{
      display:flex;
      align-items:center;
      gap:7px;
    }

    .azy-brand b{
      display:block;
      color:#fff;
      font-size:14px;
      font-weight:800;
      letter-spacing:-.01em;
    }

    .azy-live-dot{
      width:7px;
      height:7px;
      border-radius:50%;
      background:var(--azy-success);
      box-shadow:0 0 0 4px rgba(16,185,129,.11);
    }

    .azy-brand small{
      display:block;
      max-width:255px;
      margin-top:3px;
      overflow:hidden;
      color:var(--azy-muted);
      font-size:9px;
      line-height:1.5;
      text-overflow:ellipsis;
      white-space:nowrap;
    }

    .azy-header-actions{
      display:flex;
      gap:6px;
      flex:0 0 auto;
    }

    .azy-icon{
      width:32px;
      height:32px;
      padding:0;
      display:grid;
      place-items:center;
      border:1px solid var(--azy-border);
      border-radius:10px;
      background:rgba(255,255,255,.035);
      color:#cbd5e1;
      cursor:pointer;
      transition:.16s ease;
    }

    .azy-icon:hover{
      border-color:rgba(148,163,184,.32);
      background:rgba(255,255,255,.075);
      color:#fff;
      transform:translateY(-1px);
    }

    #azy-close:hover{
      border-color:rgba(239,68,68,.34);
      background:rgba(239,68,68,.11);
      color:#fecaca;
    }

    /* ---------------------------------------------------------
       Exam summary banner
    --------------------------------------------------------- */

    .azy-exam-banner{
      margin:10px 10px 0;
      padding:10px 11px;
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:10px;
      border:1px solid rgba(99,102,241,.18);
      border-radius:13px;
      background:linear-gradient(135deg,rgba(99,102,241,.09),rgba(56,189,248,.035));
    }

    .azy-exam-banner-copy{
      min-width:0;
    }

    .azy-eyebrow{
      display:block;
      margin-bottom:3px;
      color:#a5b4fc;
      font-size:8px;
      font-weight:800;
      letter-spacing:.04em;
    }

    #azy-exam-title{
      max-width:310px;
      overflow:hidden;
      color:#e2e8f0;
      font-size:10px;
      font-weight:700;
      text-overflow:ellipsis;
      white-space:nowrap;
    }

    #azy-status-chip{
      flex:0 0 auto;
      padding:5px 8px;
      border:1px solid rgba(16,185,129,.22);
      border-radius:999px;
      background:rgba(16,185,129,.08);
      color:#6ee7b7;
      font-size:8px;
      font-weight:800;
    }

    /* ---------------------------------------------------------
       Tabs
    --------------------------------------------------------- */

    .azy-tabs{
      position:relative;
      display:grid;
      grid-template-columns:repeat(4,1fr);
      gap:6px;
      padding:9px 10px 8px;
      border-bottom:1px solid var(--azy-border);
      background:rgba(2,6,23,.16);
    }

    .azy-tab{
      position:relative;
      padding:9px 7px;
      border:1px solid transparent;
      border-radius:10px;
      background:transparent;
      color:#7f8da3;
      cursor:pointer;
      font-size:10px;
      font-weight:700;
      transition:.16s ease;
    }

    .azy-tab:hover{
      color:#cbd5e1;
      background:rgba(255,255,255,.035);
    }

    .azy-tab.active{
      border-color:rgba(99,102,241,.18);
      background:linear-gradient(145deg,rgba(99,102,241,.16),rgba(139,92,246,.09));
      color:#eef2ff;
      box-shadow:inset 0 0 0 1px rgba(255,255,255,.015);
    }

    .azy-tab.active::after{
      content:"";
      position:absolute;
      right:30%;
      left:30%;
      bottom:-9px;
      height:2px;
      border-radius:3px;
      background:linear-gradient(90deg,var(--azy-primary),var(--azy-primary-2));
      box-shadow:0 0 10px rgba(99,102,241,.55);
    }

    /* ---------------------------------------------------------
       Scroll area
    --------------------------------------------------------- */

    .azy-body{
      max-height:calc(90vh - 177px);
      overflow:auto;
      padding:13px;
      scrollbar-width:thin;
      scrollbar-color:#334155 transparent;
    }

    .azy-body::-webkit-scrollbar{width:7px}
    .azy-body::-webkit-scrollbar-track{background:transparent}
    .azy-body::-webkit-scrollbar-thumb{
      border-radius:8px;
      background:#334155;
    }

    .azy-view{
      display:none;
    }

    .azy-view.active{
      display:block;
      animation:azyFade .16s ease;
    }

    @keyframes azyFade{
      from{opacity:0;transform:translateY(3px)}
      to{opacity:1;transform:translateY(0)}
    }

    /* ---------------------------------------------------------
       Stats
    --------------------------------------------------------- */

    .azy-stat-grid{
      display:grid;
      grid-template-columns:repeat(3,1fr);
      gap:8px;
    }

    .azy-stat{
      position:relative;
      overflow:hidden;
      min-height:76px;
      padding:11px 8px 10px;
      border:1px solid var(--azy-border);
      border-radius:14px;
      background:
        linear-gradient(180deg,rgba(255,255,255,.035),rgba(255,255,255,.012));
      text-align:center;
    }

    .azy-stat::before{
      content:"";
      position:absolute;
      top:0;
      right:20%;
      left:20%;
      height:1px;
      background:linear-gradient(90deg,transparent,rgba(129,140,248,.5),transparent);
    }

    .azy-stat b{
      display:block;
      color:#fff;
      font-size:21px;
      line-height:1.15;
      font-weight:850;
    }

    .azy-stat span{
      display:block;
      margin-top:5px;
      color:var(--azy-muted);
      font-size:8px;
      font-weight:600;
    }

    /* ---------------------------------------------------------
       Progress
    --------------------------------------------------------- */

    .azy-progress{
      margin-top:9px;
      padding:11px 12px;
      border:1px solid var(--azy-border);
      border-radius:14px;
      background:rgba(255,255,255,.02);
    }

    .azy-progress-row{
      display:flex;
      justify-content:space-between;
      align-items:center;
      gap:10px;
      margin-bottom:8px;
      color:#cbd5e1;
      font-size:9px;
    }

    .azy-progress-row b{
      color:#a5b4fc;
      font-size:9px;
    }

    .azy-progress-track{
      height:7px;
      overflow:hidden;
      border-radius:999px;
      background:#07101d;
      box-shadow:inset 0 0 0 1px rgba(148,163,184,.08);
    }

    #azy-progress-bar{
      position:relative;
      height:100%;
      width:0%;
      border-radius:inherit;
      background:linear-gradient(90deg,#6366f1,#8b5cf6,#10b981);
      box-shadow:0 0 14px rgba(99,102,241,.3);
      transition:width .35s cubic-bezier(.2,.8,.2,1);
    }

    #azy-progress-bar::after{
      content:"";
      position:absolute;
      inset:0;
      background:linear-gradient(100deg,transparent 30%,rgba(255,255,255,.35),transparent 70%);
      transform:translateX(-100%);
      animation:azyShimmer 2.4s ease-in-out infinite;
    }

    @keyframes azyShimmer{
      0%,55%{transform:translateX(-100%)}
      100%{transform:translateX(100%)}
    }

    /* ---------------------------------------------------------
       Sections / buttons
    --------------------------------------------------------- */

    .azy-section{
      display:flex;
      align-items:center;
      gap:8px;
      margin:15px 2px 8px;
      color:#d8e0eb;
      font-size:10px;
      font-weight:800;
    }

    .azy-section::before{
      content:"";
      width:4px;
      height:13px;
      border-radius:5px;
      background:linear-gradient(180deg,var(--azy-primary),var(--azy-primary-2));
    }

    .azy-grid{
      display:grid;
      grid-template-columns:1fr 1fr;
      gap:8px;
    }

    .azy-btn{
      min-height:39px;
      padding:9px 10px;
      display:inline-flex;
      align-items:center;
      justify-content:center;
      gap:6px;
      border:1px solid var(--azy-border);
      border-radius:11px;
      background:linear-gradient(180deg,rgba(255,255,255,.045),rgba(255,255,255,.018));
      color:#e2e8f0;
      cursor:pointer;
      font-size:10px;
      font-weight:750;
      transition:
        transform .15s ease,
        border-color .15s ease,
        background .15s ease,
        box-shadow .15s ease;
    }

    .azy-btn:hover{
      transform:translateY(-1px);
      border-color:rgba(148,163,184,.28);
      background:rgba(255,255,255,.07);
      box-shadow:0 7px 16px rgba(2,6,23,.18);
    }

    .azy-btn:active{
      transform:translateY(0) scale(.985);
    }

    .azy-primary{
      border-color:rgba(99,102,241,.4);
      background:linear-gradient(145deg,#4f46e5,#6d5dfc);
      color:#fff;
      box-shadow:0 7px 18px rgba(79,70,229,.18);
    }

    .azy-primary:hover{
      border-color:rgba(129,140,248,.62);
      background:linear-gradient(145deg,#5b52e8,#7666ff);
    }

    .azy-success{
      border-color:rgba(16,185,129,.36);
      background:linear-gradient(145deg,#047857,#059669);
      color:#ecfdf5;
      box-shadow:0 7px 18px rgba(5,150,105,.14);
    }

    .azy-success:hover{
      border-color:rgba(52,211,153,.48);
      background:linear-gradient(145deg,#058263,#0aa172);
    }

    .azy-danger{
      border-color:rgba(239,68,68,.28);
      background:rgba(127,29,29,.26);
      color:#fecaca;
    }

    .azy-wide{
      width:100%;
    }

    /* ---------------------------------------------------------
       Switches
    --------------------------------------------------------- */

    .azy-toggle{
      position:relative;
      display:flex;
      justify-content:space-between;
      align-items:center;
      gap:12px;
      margin-top:8px;
      padding:11px 12px;
      border:1px solid var(--azy-border);
      border-radius:13px;
      background:rgba(255,255,255,.018);
      cursor:pointer;
      transition:.16s ease;
    }

    .azy-toggle:hover{
      border-color:rgba(148,163,184,.25);
      background:rgba(255,255,255,.03);
    }

    .azy-toggle span{
      min-width:0;
    }

    .azy-toggle b{
      display:block;
      color:#e2e8f0;
      font-size:10px;
    }

    .azy-toggle small{
      display:block;
      margin-top:3px;
      color:#7f8da3;
      font-size:8px;
      line-height:1.55;
    }

    .azy-toggle input{
      appearance:none;
      -webkit-appearance:none;
      position:relative;
      width:38px;
      height:22px;
      flex:0 0 38px;
      margin:0;
      border:1px solid rgba(148,163,184,.2);
      border-radius:999px;
      background:#182438;
      cursor:pointer;
      transition:.18s ease;
    }

    .azy-toggle input::after{
      content:"";
      position:absolute;
      top:3px;
      right:3px;
      width:14px;
      height:14px;
      border-radius:50%;
      background:#94a3b8;
      box-shadow:0 2px 5px rgba(0,0,0,.25);
      transition:.18s ease;
    }

    .azy-toggle input:checked{
      border-color:rgba(99,102,241,.55);
      background:linear-gradient(90deg,#4f46e5,#7c3aed);
    }

    .azy-toggle input:checked::after{
      right:19px;
      background:#fff;
    }

    /* ---------------------------------------------------------
       Help / textareas
    --------------------------------------------------------- */

    .azy-help{
      margin-bottom:10px;
      padding:11px 12px;
      border:1px solid rgba(56,189,248,.15);
      border-radius:13px;
      background:linear-gradient(145deg,rgba(14,165,233,.07),rgba(99,102,241,.045));
      color:#bdd7ee;
      font-size:9px;
      line-height:1.85;
    }

    .azy-help strong{
      color:#e0f2fe;
    }

    .azy-textarea{
      width:100%;
      min-height:185px;
      padding:12px 13px;
      resize:vertical;
      border:1px solid var(--azy-border);
      border-radius:13px;
      outline:none;
      background:
        linear-gradient(180deg,rgba(2,6,23,.92),rgba(3,10,20,.92));
      color:#e2e8f0;
      caret-color:#818cf8;
      direction:ltr;
      text-align:left;
      font-family:ui-monospace,SFMono-Regular,Menlo,Monaco,Consolas,monospace!important;
      font-size:11px;
      line-height:1.75;
      box-shadow:inset 0 0 0 1px rgba(255,255,255,.01);
      transition:.16s ease;
    }

    .azy-textarea:hover{
      border-color:rgba(148,163,184,.25);
    }

    .azy-textarea:focus{
      border-color:rgba(99,102,241,.65);
      box-shadow:
        0 0 0 3px rgba(99,102,241,.10),
        inset 0 0 0 1px rgba(255,255,255,.015);
    }

    /* ---------------------------------------------------------
       Answer results
    --------------------------------------------------------- */

    #azy-results{
      display:grid;
      grid-template-columns:repeat(5,1fr);
      gap:6px;
      margin-top:11px;
    }

    .azy-result{
      min-height:46px;
      padding:7px 4px;
      display:grid;
      place-items:center;
      gap:2px;
      border:1px solid var(--azy-border);
      border-radius:10px;
      background:rgba(255,255,255,.02);
      text-align:center;
      font-size:8px;
    }

    .azy-result b{
      display:block;
      color:#cbd5e1;
      font-size:10px;
    }

    .azy-result.correct{
      border-color:rgba(16,185,129,.28);
      background:rgba(16,185,129,.075);
      color:#6ee7b7;
    }

    .azy-result.wrong{
      border-color:rgba(239,68,68,.28);
      background:rgba(239,68,68,.075);
      color:#fca5a5;
    }

    .azy-result.empty{
      border-color:rgba(245,158,11,.28);
      background:rgba(245,158,11,.07);
      color:#fcd34d;
    }

    .azy-result.nokey{
      color:#94a3b8;
    }

    .azy-note{
      margin-top:10px;
      padding:8px 10px;
      border:1px dashed rgba(148,163,184,.13);
      border-radius:10px;
      color:#64748b;
      text-align:center;
      font-size:8px;
      line-height:1.7;
    }

    /* ---------------------------------------------------------
       Footer
    --------------------------------------------------------- */

    .azy-footer{
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:8px;
      margin-top:11px;
      padding-top:9px;
      border-top:1px solid var(--azy-border);
      color:#536176;
      font-size:8px;
    }

    .azy-footer-secure{
      display:flex;
      align-items:center;
      gap:5px;
    }

    .azy-footer-secure::before{
      content:"";
      width:6px;
      height:6px;
      border-radius:50%;
      background:var(--azy-success);
      box-shadow:0 0 0 3px rgba(16,185,129,.08);
    }

    /* ---------------------------------------------------------
       Settings
    --------------------------------------------------------- */

    .azy-settings-group{
      margin-bottom:12px;
      overflow:hidden;
      border:1px solid var(--azy-border);
      border-radius:15px;
      background:rgba(255,255,255,.015);
    }

    .azy-settings-head{
      padding:11px 12px 9px;
      border-bottom:1px solid var(--azy-border);
      background:linear-gradient(180deg,rgba(255,255,255,.025),transparent);
    }

    .azy-settings-head b{
      display:block;
      color:#e2e8f0;
      font-size:10px;
      font-weight:850;
    }

    .azy-settings-head span{
      display:block;
      margin-top:3px;
      color:#64748b;
      font-size:8px;
      line-height:1.6;
    }

    .azy-settings-body{
      padding:4px 10px 10px;
    }

    .azy-settings-body .azy-toggle{
      margin-top:7px;
      border-color:rgba(148,163,184,.11);
      background:rgba(2,6,23,.15);
    }

    .azy-settings-actions{
      display:grid;
      grid-template-columns:1fr 1fr;
      gap:8px;
      padding:10px;
    }

    .azy-storage-row{
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:10px;
      padding:10px 12px;
      border-top:1px solid var(--azy-border);
      color:#94a3b8;
      font-size:8px;
    }

    .azy-storage-row strong{
      color:#cbd5e1;
      font-size:9px;
    }

    .azy-privacy-card{
      padding:11px 12px;
      display:flex;
      align-items:flex-start;
      gap:9px;
      border:1px solid rgba(16,185,129,.13);
      border-radius:13px;
      background:rgba(16,185,129,.045);
      color:#8ecfb9;
      font-size:8px;
      line-height:1.75;
    }

    .azy-privacy-icon{
      width:28px;
      height:28px;
      flex:0 0 28px;
      display:grid;
      place-items:center;
      border-radius:9px;
      background:rgba(16,185,129,.09);
      color:#6ee7b7;
      font-size:13px;
      font-weight:900;
    }

    /* ---------------------------------------------------------
       Onboarding
    --------------------------------------------------------- */

    .azy-onboarding-modal{
      background:
        radial-gradient(circle at 50% 20%,rgba(99,102,241,.13),transparent 36%),
        rgba(2,6,23,.78);
      backdrop-filter:blur(12px);
      -webkit-backdrop-filter:blur(12px);
    }

    .azy-onboarding-card{
      width:min(430px,100%);
      max-height:calc(100vh - 24px);
      overflow:auto;
      scrollbar-width:thin;
      scrollbar-color:#334155 transparent;
      border:1px solid rgba(148,163,184,.24);
      border-radius:22px;
      background:
        radial-gradient(circle at 100% 0%,rgba(99,102,241,.12),transparent 32%),
        linear-gradient(180deg,#0f1a2d,#081321);
      box-shadow:0 34px 100px rgba(0,0,0,.58);
      transform:translateY(14px) scale(.975);
      transition:transform .22s cubic-bezier(.2,.8,.2,1);
    }

    .azy-modal.show .azy-onboarding-card{
      transform:translateY(0) scale(1);
    }

    .azy-onboarding-top{
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:10px;
      padding:12px 13px 10px;
      border-bottom:1px solid var(--azy-border);
    }

    .azy-onboarding-brand{
      display:flex;
      align-items:center;
      gap:8px;
      min-width:0;
    }

    .azy-onboarding-mini-logo{
      width:32px;
      height:32px;
      flex:0 0 32px;
      display:grid;
      place-items:center;
      border-radius:10px;
      background:linear-gradient(145deg,var(--azy-primary),var(--azy-primary-2));
      color:#fff;
      font-size:15px;
      font-weight:900;
      box-shadow:0 7px 16px rgba(79,70,229,.25);
    }

    .azy-onboarding-brand b{
      display:block;
      color:#fff;
      font-size:11px;
    }

    .azy-onboarding-brand small{
      display:block;
      margin-top:2px;
      color:#6f7d91;
      font-size:8px;
    }

    .azy-onboarding-step-count{
      flex:0 0 auto;
      padding:5px 8px;
      border:1px solid rgba(99,102,241,.22);
      border-radius:999px;
      background:rgba(99,102,241,.08);
      color:#c7d2fe;
      font-size:8px;
      font-weight:800;
    }

    .azy-onboarding-progress{
      height:3px;
      background:#07101d;
    }

    #azy-onboarding-progress-bar{
      width:20%;
      height:100%;
      background:linear-gradient(90deg,#6366f1,#8b5cf6,#10b981);
      transition:width .25s cubic-bezier(.2,.8,.2,1);
    }

    .azy-onboarding-content{
      min-height:340px;
      padding:20px 20px 14px;
    }

    .azy-onboarding-step{
      display:none;
      animation:azyOnboardIn .22s ease;
    }

    .azy-onboarding-step.active{
      display:block;
    }

    @keyframes azyOnboardIn{
      from{opacity:0;transform:translateY(6px)}
      to{opacity:1;transform:translateY(0)}
    }

    .azy-onboarding-visual{
      width:72px;
      height:72px;
      margin:0 auto 17px;
      display:grid;
      place-items:center;
      border:1px solid rgba(99,102,241,.22);
      border-radius:22px;
      background:
        radial-gradient(circle at 30% 25%,rgba(255,255,255,.14),transparent 30%),
        linear-gradient(145deg,rgba(99,102,241,.18),rgba(139,92,246,.08));
      color:#c7d2fe;
      font-size:28px;
      font-weight:900;
      box-shadow:inset 0 0 0 1px rgba(255,255,255,.018);
    }

    .azy-onboarding-step h2{
      margin:0;
      color:#fff;
      text-align:center;
      font-size:17px;
      line-height:1.5;
      font-weight:900;
    }

    .azy-onboarding-step > p{
      max-width:340px;
      margin:8px auto 0;
      color:#94a3b8;
      text-align:center;
      font-size:9px;
      line-height:1.9;
    }

    .azy-onboarding-points{
      display:grid;
      gap:7px;
      margin-top:17px;
    }

    .azy-onboarding-point{
      display:flex;
      align-items:flex-start;
      gap:8px;
      padding:9px 10px;
      border:1px solid rgba(148,163,184,.11);
      border-radius:11px;
      background:rgba(255,255,255,.018);
      color:#a9b5c7;
      font-size:8px;
      line-height:1.7;
    }

    .azy-onboarding-point i{
      width:20px;
      height:20px;
      flex:0 0 20px;
      display:grid;
      place-items:center;
      border-radius:7px;
      background:rgba(99,102,241,.09);
      color:#a5b4fc;
      font-style:normal;
      font-size:9px;
      font-weight:900;
    }

    .azy-onboarding-dots{
      display:flex;
      align-items:center;
      justify-content:center;
      gap:5px;
      padding:0 14px 11px;
    }

    .azy-onboarding-dot{
      width:6px;
      height:6px;
      border-radius:999px;
      background:#26354b;
      transition:.18s ease;
    }

    .azy-onboarding-dot.active{
      width:20px;
      background:linear-gradient(90deg,var(--azy-primary),var(--azy-primary-2));
      box-shadow:0 0 10px rgba(99,102,241,.3);
    }

    .azy-onboarding-actions{
      display:grid;
      grid-template-columns:1fr 1.45fr 1fr;
      gap:7px;
      padding:10px 11px;
      border-top:1px solid var(--azy-border);
      background:rgba(2,6,23,.24);
    }

    .azy-onboarding-secondary{
      display:flex;
      justify-content:center;
      gap:14px;
      padding:0 10px 11px;
      background:rgba(2,6,23,.24);
    }

    .azy-link-btn{
      padding:3px;
      border:0;
      background:transparent;
      color:#64748b;
      cursor:pointer;
      font-size:8px;
    }

    .azy-link-btn:hover{
      color:#cbd5e1;
      text-decoration:underline;
    }

    /* ---------------------------------------------------------
       Modal
    --------------------------------------------------------- */

    .azy-modal{
      position:fixed;
      inset:0;
      z-index:2147483647;
      display:flex;
      align-items:center;
      justify-content:center;
      padding:18px;
      opacity:0;
      visibility:hidden;
      pointer-events:none;
      background:rgba(2,6,23,.72);
      backdrop-filter:blur(8px);
      -webkit-backdrop-filter:blur(8px);
      transition:
        opacity .18s ease,
        visibility .18s ease;
    }

    .azy-modal.show{
      opacity:1;
      visibility:visible;
      pointer-events:auto;
    }

    .azy-modal-card{
      width:min(380px,100%);
      overflow:hidden;
      border:1px solid rgba(148,163,184,.22);
      border-radius:20px;
      background:
        radial-gradient(circle at 100% 0%,rgba(239,68,68,.07),transparent 32%),
        linear-gradient(180deg,#0e192b,#091321);
      box-shadow:0 30px 90px rgba(0,0,0,.55);
      transform:translateY(12px) scale(.975);
      transition:transform .2s cubic-bezier(.2,.8,.2,1);
    }

    .azy-modal.show .azy-modal-card{
      transform:translateY(0) scale(1);
    }

    .azy-modal-content{
      padding:18px 18px 14px;
    }

    .azy-modal-icon{
      width:46px;
      height:46px;
      display:grid;
      place-items:center;
      margin-bottom:13px;
      border:1px solid rgba(239,68,68,.22);
      border-radius:14px;
      background:rgba(239,68,68,.09);
      color:#fca5a5;
      font-size:22px;
      font-weight:800;
    }

    .azy-modal-title{
      margin:0;
      color:#fff;
      font-size:15px;
      font-weight:850;
    }

    .azy-modal-description{
      margin:7px 0 0;
      color:#94a3b8;
      font-size:10px;
      line-height:1.85;
    }

    .azy-modal-meta{
      margin-top:12px;
      padding:9px 10px;
      display:flex;
      align-items:center;
      gap:7px;
      border:1px solid rgba(16,185,129,.13);
      border-radius:10px;
      background:rgba(16,185,129,.045);
      color:#86d9bd;
      font-size:8px;
      line-height:1.6;
    }

    .azy-modal-actions{
      display:grid;
      grid-template-columns:1fr 1fr;
      gap:8px;
      padding:11px;
      border-top:1px solid var(--azy-border);
      background:rgba(2,6,23,.2);
    }

    .azy-modal-danger{
      border-color:rgba(239,68,68,.35);
      background:linear-gradient(145deg,#991b1b,#b91c1c);
      color:#fff;
    }

    .azy-modal-danger:hover{
      border-color:rgba(248,113,113,.55);
      background:linear-gradient(145deg,#a51d1d,#c22222);
    }

    /* ---------------------------------------------------------
       Toast
    --------------------------------------------------------- */

    #azy-toast{
      position:fixed;
      right:24px;
      bottom:96px;
      width:min(340px,calc(100vw - 32px));
      z-index:2147483647;
      opacity:0;
      visibility:hidden;
      pointer-events:none;
      transform:translateY(12px) scale(.985);
      transition:
        opacity .2s ease,
        transform .2s cubic-bezier(.2,.8,.2,1),
        visibility .2s ease;
    }

    #azy-toast.show{
      opacity:1;
      visibility:visible;
      pointer-events:auto;
      transform:translateY(0) scale(1);
    }

    .azy-toast-card{
      position:relative;
      overflow:hidden;
      display:flex;
      align-items:flex-start;
      gap:10px;
      padding:11px 12px 12px;
      border:1px solid var(--azy-border-strong);
      border-radius:14px;
      background:rgba(8,17,31,.97);
      box-shadow:0 18px 45px rgba(2,6,23,.42);
      backdrop-filter:blur(16px);
    }

    .azy-toast-icon{
      width:30px;
      height:30px;
      flex:0 0 30px;
      display:grid;
      place-items:center;
      border-radius:9px;
      font-size:14px;
      font-weight:900;
    }

    .azy-toast-copy{
      min-width:0;
      flex:1;
      padding-top:1px;
    }

    .azy-toast-title{
      display:block;
      color:#f8fafc;
      font-size:10px;
      font-weight:800;
    }

    .azy-toast-message{
      display:block;
      margin-top:2px;
      color:#a8b4c6;
      font-size:9px;
      line-height:1.55;
    }

    .azy-toast-close{
      width:24px;
      height:24px;
      padding:0;
      display:grid;
      place-items:center;
      border:0;
      border-radius:7px;
      background:transparent;
      color:#64748b;
      cursor:pointer;
      font-size:14px;
    }

    .azy-toast-close:hover{
      background:rgba(255,255,255,.05);
      color:#cbd5e1;
    }

    .azy-toast-progress{
      position:absolute;
      right:0;
      bottom:0;
      left:0;
      height:2px;
      transform-origin:right;
      animation:azyToastTimer var(--azy-toast-duration,3000ms) linear forwards;
    }

    @keyframes azyToastTimer{
      from{transform:scaleX(1)}
      to{transform:scaleX(0)}
    }

    #azy-toast[data-type="success"] .azy-toast-card{
      border-color:rgba(16,185,129,.26);
    }

    #azy-toast[data-type="success"] .azy-toast-icon{
      background:rgba(16,185,129,.11);
      color:#6ee7b7;
    }

    #azy-toast[data-type="success"] .azy-toast-progress{
      background:var(--azy-success);
    }

    #azy-toast[data-type="warning"] .azy-toast-card{
      border-color:rgba(245,158,11,.27);
    }

    #azy-toast[data-type="warning"] .azy-toast-icon{
      background:rgba(245,158,11,.11);
      color:#fcd34d;
    }

    #azy-toast[data-type="warning"] .azy-toast-progress{
      background:var(--azy-warning);
    }

    #azy-toast[data-type="error"] .azy-toast-card{
      border-color:rgba(239,68,68,.28);
    }

    #azy-toast[data-type="error"] .azy-toast-icon{
      background:rgba(239,68,68,.11);
      color:#fca5a5;
    }

    #azy-toast[data-type="error"] .azy-toast-progress{
      background:var(--azy-danger);
    }

    #azy-toast[data-type="info"] .azy-toast-card{
      border-color:rgba(56,189,248,.24);
    }

    #azy-toast[data-type="info"] .azy-toast-icon{
      background:rgba(56,189,248,.10);
      color:#7dd3fc;
    }

    #azy-toast[data-type="info"] .azy-toast-progress{
      background:var(--azy-info);
    }

    /* ---------------------------------------------------------
       Responsive
    --------------------------------------------------------- */

    @media(max-width:520px){
      #azy-bubble{
        right:14px;
        bottom:14px;
        width:58px;
        height:58px;
      }

      #azy-panel{
        right:6px;
        bottom:6px;
        width:calc(100vw - 12px);
        max-height:95vh;
        border-radius:18px;
      }

      .azy-body{
        max-height:calc(95vh - 177px);
        padding:11px;
      }

      .azy-header{
        min-height:66px;
        padding:11px 12px;
      }

      .azy-exam-banner{
        margin:8px 8px 0;
      }

      #azy-results{
        grid-template-columns:repeat(4,1fr);
      }

      #azy-toast{
        right:12px;
        bottom:82px;
        width:calc(100vw - 24px);
      }
    }

    @media(max-width:360px){
      .azy-grid{
        grid-template-columns:1fr;
      }

      .azy-stat-grid{
        gap:5px;
      }

      .azy-stat{
        min-height:70px;
        padding-inline:5px;
      }

      #azy-results{
        grid-template-columns:repeat(3,1fr);
      }
    }

    @media(prefers-reduced-motion:reduce){
      #${APP_ID} *,
      #${APP_ID} *::before,
      #${APP_ID} *::after{
        animation-duration:.001ms!important;
        animation-iteration-count:1!important;
        transition-duration:.001ms!important;
      }
    }
`;

  document.head.appendChild(
    style
  );

  // ============================================================
  // UI
  // ============================================================

  const root =
    document.createElement(
      "div"
    );

  root.id =
    APP_ID;

  root.innerHTML = `
    <button
      id="azy-bubble"
      type="button"
      aria-label="باز کردن آزمونیار"
      title="آزمونیار"
    >
      آ
      <span id="azy-badge">0</span>
    </button>

    <section
      id="azy-panel"
      class="azy-hidden"
      role="dialog"
      aria-label="آزمونیار"
    >
      <header class="azy-header">
        <div class="azy-brand">
          <div class="azy-logo" aria-hidden="true">آ</div>

          <div class="azy-brand-copy">
            <div class="azy-brand-title-row">
              <b>آزمونیار</b>
              <span class="azy-live-dot" title="فعال"></span>
            </div>

            <small>دستیار استخراج و پاسخ آزمونهای تمرینی</small>
          </div>
        </div>

        <div class="azy-header-actions">
          <button
            class="azy-icon"
            id="azy-minimize"
            type="button"
            title="کوچک کردن"
            aria-label="کوچک کردن"
          >
            −
          </button>

          <button
            class="azy-icon"
            id="azy-close"
            type="button"
            title="بستن"
            aria-label="بستن آزمونیار"
          >
            ×
          </button>
        </div>
      </header>

      <div class="azy-exam-banner">
        <div class="azy-exam-banner-copy">
          <span class="azy-eyebrow">آزمون فعال</span>
          <div id="azy-exam-title">در حال شناسایی...</div>
        </div>

        <span id="azy-status-chip">فعال</span>
      </div>

      <nav class="azy-tabs" aria-label="بخشهای آزمونیار">
        <button
          class="azy-tab active"
          type="button"
          data-tab="home"
        >
          ◈ خانه
        </button>

        <button
          class="azy-tab"
          type="button"
          data-tab="export"
        >
          ⇩ خروجی
        </button>

        <button
          class="azy-tab"
          type="button"
          data-tab="answers"
        >
          ✓ پاسخها
        </button>

        <button
          class="azy-tab"
          type="button"
          data-tab="settings"
        >
          ⚙ تنظیمات
        </button>
      </nav>

      <main class="azy-body">

        <!-- HOME -->
        <section
          class="azy-view active"
          data-view="home"
        >
          <div class="azy-stat-grid">
            <div class="azy-stat">
              <b id="azy-current">0</b>
              <span>سؤال در صفحه</span>
            </div>

            <div class="azy-stat">
              <b id="azy-saved">0</b>
              <span>ذخیرهشده</span>
            </div>

            <div class="azy-stat">
              <b id="azy-expected">0</b>
              <span>کل آزمون</span>
            </div>
          </div>

          <div class="azy-progress">
            <div class="azy-progress-row">
              <span>پیشرفت جمعآوری سؤالها</span>
              <b id="azy-progress-text">0 / 0</b>
            </div>

            <div class="azy-progress-track">
              <div id="azy-progress-bar"></div>
            </div>
          </div>

          <div class="azy-section">استخراج و خروجی</div>

          <div class="azy-grid">
            <button
              class="azy-btn azy-primary"
              id="azy-scan"
              type="button"
            >
              ↻ اسکن همه سؤالها
            </button>

            <button
              class="azy-btn"
              id="azy-copy"
              type="button"
            >
              ⧉ کپی برای ChatGPT
            </button>

            <button
              class="azy-btn azy-success"
              id="azy-download"
              type="button"
            >
              ↓ دانلود TXT
            </button>

            <button
              class="azy-btn"
              id="azy-preview"
              type="button"
            >
              ◫ پیشنمایش خروجی
            </button>
          </div>

          <label class="azy-toggle">
            <span>
              <b>ذخیره خودکار سؤالها</b>
              <small>
                سؤالهای جدیدی که وارد صفحه میشوند بهصورت خودکار ذخیره شوند.
              </small>
            </span>

            <input
              type="checkbox"
              id="azy-auto-capture"
              aria-label="ذخیره خودکار سؤالها"
            >
          </label>

          <div class="azy-section">پاسخدهی</div>

          <div class="azy-grid">
            <button
              class="azy-btn azy-success"
              id="azy-fill"
              type="button"
            >
              ✓ پر کردن پاسخها
            </button>

            <button
              class="azy-btn"
              id="azy-check"
              type="button"
            >
              ◎ بررسی انتخابها
            </button>
          </div>

          <label class="azy-toggle">
            <span>
              <b>Auto Fill</b>
              <small>
                بعد از ذخیره کلید، پاسخ سؤالهای موجود خودکار انتخاب شوند.
              </small>
            </span>

            <input
              type="checkbox"
              id="azy-auto-fill"
              aria-label="Auto Fill"
            >
          </label>

          <div class="azy-note">
            آزمونیار Submit یا Finish Attempt را خودکار انجام نمیدهد.
          </div>

          <div class="azy-footer">
            <span>Console Edition v3.2.0</span>
            <span class="azy-footer-secure">ذخیره محلی روی مرورگر</span>
          </div>
        </section>

        <!-- EXPORT -->
        <section
          class="azy-view"
          data-view="export"
        >
          <div class="azy-help">
            <strong>خروجی آماده است.</strong>
            این متن شامل Prompt مخصوص ChatGPT، تمام سؤالهای ذخیرهشده
            و گزینههای آنهاست. میتوانی آن را مستقیم Copy کنی یا بهصورت TXT دانلود کنی.
          </div>

          <textarea
            id="azy-preview-text"
            class="azy-textarea"
            style="min-height:330px"
            readonly
            spellcheck="false"
            aria-label="پیشنمایش خروجی"
          ></textarea>

          <div
            class="azy-grid"
            style="margin-top:8px"
          >
            <button
              class="azy-btn azy-primary"
              id="azy-copy-2"
              type="button"
            >
              ⧉ کپی کامل
            </button>

            <button
              class="azy-btn azy-success"
              id="azy-download-2"
              type="button"
            >
              ↓ دانلود فایل TXT
            </button>
          </div>

          <div class="azy-footer">
            <span>Prompt + Questions</span>
            <span class="azy-footer-secure">آماده ارسال</span>
          </div>
        </section>

        <!-- ANSWERS -->
        <section
          class="azy-view"
          data-view="answers"
        >
          <div class="azy-help">
            <strong>کلید پاسخ ChatGPT را اینجا Paste کن.</strong><br>
            فرمت پیشنهادی:
            <b>1. D</b>، <b>2. B</b>، <b>3. A</b>.
            فرمت کوتاه مثل <b>DBAC...</b> و JSON نیز پشتیبانی میشود.
          </div>

          <textarea
            id="azy-answer-key"
            class="azy-textarea"
            placeholder="1. D&#10;2. B&#10;3. A"
            spellcheck="false"
            aria-label="کلید پاسخ"
          ></textarea>

          <div
            class="azy-grid"
            style="margin-top:8px"
          >
            <button
              class="azy-btn azy-primary"
              id="azy-save-key"
              type="button"
            >
              ✓ ذخیره کلید پاسخ
            </button>

            <button
              class="azy-btn azy-success"
              id="azy-fill-2"
              type="button"
            >
              ↯ پر کردن پاسخها
            </button>
          </div>

          <div id="azy-results"></div>

          <div class="azy-footer">
            <span>Answer Key</span>
            <span class="azy-footer-secure">بدون ارسال خودکار</span>
          </div>
        </section>

        <!-- SETTINGS -->
        <section
          class="azy-view"
          data-view="settings"
        >
          <div class="azy-settings-group">
            <div class="azy-settings-head">
              <b>رفتار آزمونیار</b>
              <span>نحوه جمعآوری سؤالها، پاسخدهی و نمایش پنل را تنظیم کن.</span>
            </div>

            <div class="azy-settings-body">
              <label class="azy-toggle">
                <span>
                  <b>ذخیره خودکار سؤالها</b>
                  <small>هر سؤال جدیدی که وارد صفحه شود بهصورت خودکار ذخیره شود.</small>
                </span>

                <input
                  type="checkbox"
                  id="azy-settings-auto-capture"
                  aria-label="ذخیره خودکار سؤالها در تنظیمات"
                >
              </label>

              <label class="azy-toggle">
                <span>
                  <b>Auto Fill</b>
                  <small>اگر کلید پاسخ ذخیره شده باشد، پاسخ سؤالهای موجود خودکار انتخاب شوند.</small>
                </span>

                <input
                  type="checkbox"
                  id="azy-settings-auto-fill"
                  aria-label="Auto Fill در تنظیمات"
                >
              </label>

              <label class="azy-toggle">
                <span>
                  <b>باز شدن پنل در شروع</b>
                  <small>پس از اجرای اسکریپت، بهجای حباب کوچک، پنل کامل باز شود.</small>
                </span>

                <input
                  type="checkbox"
                  id="azy-open-on-start"
                  aria-label="باز شدن پنل در شروع"
                >
              </label>

              <label class="azy-toggle">
                <span>
                  <b>اعلانهای Toast</b>
                  <small>پیامهای موفقیت، هشدار و خطا در پایین صفحه نمایش داده شوند.</small>
                </span>

                <input
                  type="checkbox"
                  id="azy-notifications"
                  aria-label="اعلانهای Toast"
                >
              </label>
            </div>
          </div>

          <div class="azy-settings-group">
            <div class="azy-settings-head">
              <b>راهنما و Onboarding</b>
              <span>آموزش را دوباره ببین یا تنظیم کن که در اجرای بعدی نمایش داده شود.</span>
            </div>

            <div class="azy-settings-actions">
              <button
                class="azy-btn azy-primary"
                id="azy-show-onboarding"
                type="button"
              >
                ◈ نمایش آموزش
              </button>

              <button
                class="azy-btn"
                id="azy-reset-onboarding"
                type="button"
              >
                ↻ نمایش در اجرای بعد
              </button>
            </div>
          </div>

          <div class="azy-settings-group">
            <div class="azy-settings-head">
              <b>دادهها و پشتیبان</b>
              <span>یک نسخه JSON از دادههای آزمونیار بگیر یا فقط اطلاعات همین آزمون را پاک کن.</span>
            </div>

            <div class="azy-settings-actions">
              <button
                class="azy-btn"
                id="azy-backup-data"
                type="button"
              >
                ↓ پشتیبان JSON
              </button>

              <button
                class="azy-btn azy-danger"
                id="azy-clear-current-exam"
                type="button"
              >
                ⌫ پاک کردن این آزمون
              </button>
            </div>

            <div class="azy-storage-row">
              <span>حجم ذخیره محلی</span>
              <strong id="azy-storage-size">0 KB</strong>
            </div>
          </div>

          <div class="azy-privacy-card">
            <div class="azy-privacy-icon">✓</div>

            <div>
              <strong style="display:block;color:#bbf7d0;font-size:9px;margin-bottom:3px">
                دادهها روی همین مرورگر میمانند
              </strong>
              سؤالها، کلید پاسخ و تنظیمات با localStorage ذخیره میشوند.
              آزمونیار در این نسخه اطلاعات را به سرور خارجی ارسال نمیکند و
              Submit / Finish Attempt را نیز خودکار انجام نمیدهد.
            </div>
          </div>

          <div class="azy-footer">
            <span>Console Edition v3.2.0</span>
            <span class="azy-footer-secure">Settings + Onboarding</span>
          </div>
        </section>

      </main>
    </section>

    <!-- Toast -->
    <div
      id="azy-toast"
      data-type="info"
      role="status"
      aria-live="polite"
    >
      <div class="azy-toast-card">
        <div class="azy-toast-icon" id="azy-toast-icon">i</div>

        <div class="azy-toast-copy">
          <span class="azy-toast-title" id="azy-toast-title">آزمونیار</span>
          <span class="azy-toast-message" id="azy-toast-message"></span>
        </div>

        <button
          class="azy-toast-close"
          id="azy-toast-close"
          type="button"
          aria-label="بستن اعلان"
        >
          ×
        </button>

        <div class="azy-toast-progress"></div>
      </div>
    </div>

    <!-- Onboarding Modal -->
    <div
      class="azy-modal azy-onboarding-modal"
      id="azy-onboarding-modal"
      role="presentation"
    >
      <div
        class="azy-onboarding-card"
        role="dialog"
        aria-modal="true"
        aria-labelledby="azy-onboarding-title"
      >
        <div class="azy-onboarding-top">
          <div class="azy-onboarding-brand">
            <div class="azy-onboarding-mini-logo" aria-hidden="true">آ</div>

            <div>
              <b>آزمونیار</b>
              <small>شروع سریع و مرحلهبهمرحله</small>
            </div>
          </div>

          <span class="azy-onboarding-step-count" id="azy-onboarding-count">
            1 / 5
          </span>
        </div>

        <div class="azy-onboarding-progress">
          <div id="azy-onboarding-progress-bar"></div>
        </div>

        <div class="azy-onboarding-content">
          <section
            class="azy-onboarding-step active"
            data-onboarding-step="1"
          >
            <div class="azy-onboarding-visual">آ</div>

            <h2 id="azy-onboarding-title">آزمونیار چیست؟</h2>

            <p>
              یک دستیار محلی برای آزمونهای تمرینی Moodle است که سؤالها را جمع میکند،
              برای ChatGPT خروجی آماده میسازد و کلید پاسخ را دوباره داخل آزمون استفاده میکند.
            </p>

            <div class="azy-onboarding-points">
              <div class="azy-onboarding-point">
                <i>1</i>
                <span>سؤالها و گزینههای موجود در صفحه شناسایی و ذخیره میشوند.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>2</i>
                <span>همه اطلاعات در localStorage همین مرورگر باقی میمانند.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>3</i>
                <span>دکمه Submit یا Finish Attempt هیچوقت خودکار زده نمیشود.</span>
              </div>
            </div>
          </section>

          <section
            class="azy-onboarding-step"
            data-onboarding-step="2"
          >
            <div class="azy-onboarding-visual">↻</div>

            <h2>مرحله ۱: سؤالها را جمع کن</h2>

            <p>
              آزمونیار سؤالهای موجود در DOM را میخواند. اگر آزمون چندصفحه باشد،
              هنگام رفتن به سؤال یا صفحه بعدی، سؤالهای جدید به ذخیره قبلی اضافه میشوند.
            </p>

            <div class="azy-onboarding-points">
              <div class="azy-onboarding-point">
                <i>✓</i>
                <span>عدد «ذخیرهشده / کل آزمون» نشان میدهد چند سؤال جمع شده است.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>↻</i>
                <span>«اسکن همه سؤالها» هر زمان خواستی یک اسکن دستی انجام میدهد.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>A</i>
                <span>گزینههای A/B/C/D همراه با متن خود سؤال ذخیره میشوند.</span>
              </div>
            </div>
          </section>

          <section
            class="azy-onboarding-step"
            data-onboarding-step="3"
          >
            <div class="azy-onboarding-visual">⇩</div>

            <h2>مرحله ۲: برای ChatGPT خروجی بگیر</h2>

            <p>
              وقتی سؤالها کامل شدند، از تب خروجی استفاده کن. فایل TXT علاوه بر سؤالها،
              Prompt آماده دارد تا پاسخ دقیقا به فرمت قابل استفاده آزمونیار برگردد.
            </p>

            <div class="azy-onboarding-points">
              <div class="azy-onboarding-point">
                <i>⧉</i>
                <span>«کپی برای ChatGPT» همه Prompt + Questions را Clipboard میکند.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>↓</i>
                <span>«دانلود TXT» همان محتوا را در یک فایل قابل Upload ذخیره میکند.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>50</i>
                <span>قبل از ارسال، بهتر است تعداد ذخیرهشده با تعداد کل آزمون برابر باشد.</span>
              </div>
            </div>
          </section>

          <section
            class="azy-onboarding-step"
            data-onboarding-step="4"
          >
            <div class="azy-onboarding-visual">✓</div>

            <h2>مرحله ۳: کلید پاسخ را برگردان</h2>

            <p>
              پاسخ ChatGPT را در تب «پاسخها» Paste کن، کلید را ذخیره کن و سپس
              پاسخهای موجود در صفحه را پر یا بررسی کن.
            </p>

            <div class="azy-onboarding-points">
              <div class="azy-onboarding-point">
                <i>1</i>
                <span>فرمت پیشنهادی: 1. D ، 2. B ، 3. A و به همین ترتیب.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>↯</i>
                <span>Auto Fill فقط گزینهها را انتخاب میکند؛ ارسال نهایی با خود کاربر است.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>◎</i>
                <span>«بررسی انتخابها» انتخاب فعلی را با کلید ذخیرهشده مقایسه میکند.</span>
              </div>
            </div>
          </section>

          <section
            class="azy-onboarding-step"
            data-onboarding-step="5"
          >
            <div class="azy-onboarding-visual">⚙</div>

            <h2>همه چیز آماده است</h2>

            <p>
              در Settings میتوانی Auto Capture، Auto Fill، باز شدن پنل و اعلانها را تغییر دهی،
              آموزش را دوباره اجرا کنی یا از دادهها پشتیبان بگیری.
            </p>

            <div class="azy-onboarding-points">
              <div class="azy-onboarding-point">
                <i>⚙</i>
                <span>تنظیمات برای دفعات بعدی در همین مرورگر ذخیره میشوند.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>↓</i>
                <span>پشتیبان JSON شامل تنظیمات و همه آزمونهای ذخیرهشده است.</span>
              </div>

              <div class="azy-onboarding-point">
                <i>✓</i>
                <span>حالا فقط آزمون را جلو برو؛ جمعآوری میتواند خودکار انجام شود.</span>
              </div>
            </div>
          </section>
        </div>

        <div class="azy-onboarding-dots">
          <span class="azy-onboarding-dot active" data-onboarding-dot="1"></span>
          <span class="azy-onboarding-dot" data-onboarding-dot="2"></span>
          <span class="azy-onboarding-dot" data-onboarding-dot="3"></span>
          <span class="azy-onboarding-dot" data-onboarding-dot="4"></span>
          <span class="azy-onboarding-dot" data-onboarding-dot="5"></span>
        </div>

        <div class="azy-onboarding-actions">
          <button
            class="azy-btn"
            id="azy-onboarding-prev"
            type="button"
          >
            قبلی
          </button>

          <button
            class="azy-btn azy-primary"
            id="azy-onboarding-next"
            type="button"
          >
            بعدی
          </button>

          <button
            class="azy-btn"
            id="azy-onboarding-later"
            type="button"
          >
            بعداً
          </button>
        </div>

        <div class="azy-onboarding-secondary">
          <button
            class="azy-link-btn"
            id="azy-onboarding-never"
            type="button"
          >
            دیگر خودکار نشان نده
          </button>
        </div>
      </div>
    </div>

    <!-- Data Confirm Modal -->
    <div
      class="azy-modal"
      id="azy-data-modal"
      role="presentation"
    >
      <div
        class="azy-modal-card"
        role="alertdialog"
        aria-modal="true"
        aria-labelledby="azy-data-modal-title"
        aria-describedby="azy-data-modal-description"
      >
        <div class="azy-modal-content">
          <div class="azy-modal-icon" aria-hidden="true">!</div>

          <h3
            class="azy-modal-title"
            id="azy-data-modal-title"
          >
            اطلاعات این آزمون پاک شود؟
          </h3>

          <p
            class="azy-modal-description"
            id="azy-data-modal-description"
          >
            سؤالهای ذخیرهشده و کلید پاسخ همین آزمون حذف میشوند.
            تنظیمات کلی آزمونیار دستنخورده باقی میمانند.
          </p>

          <div class="azy-modal-meta">
            این کار فقط روی آزمون فعلی انجام میشود.
          </div>
        </div>

        <div class="azy-modal-actions">
          <button
            class="azy-btn"
            id="azy-data-cancel"
            type="button"
          >
            انصراف
          </button>

          <button
            class="azy-btn azy-modal-danger"
            id="azy-data-confirm"
            type="button"
          >
            بله، پاک کن
          </button>
        </div>
      </div>
    </div>

    <!-- Close Modal -->
    <div
      class="azy-modal"
      id="azy-close-modal"
      role="presentation"
    >
      <div
        class="azy-modal-card"
        role="alertdialog"
        aria-modal="true"
        aria-labelledby="azy-close-modal-title"
        aria-describedby="azy-close-modal-description"
      >
        <div class="azy-modal-content">
          <div class="azy-modal-icon" aria-hidden="true">×</div>

          <h3
            class="azy-modal-title"
            id="azy-close-modal-title"
          >
            آزمونیار بسته شود؟
          </h3>

          <p
            class="azy-modal-description"
            id="azy-close-modal-description"
          >
            پنل و حباب از صفحه مخفی میشوند؛ سؤالها، کلید پاسخ و تنظیمات
            ذخیرهشده پاک نخواهند شد.
          </p>

          <div class="azy-modal-meta">
            ✓ برای بازگرداندن نسخه Console، کافی است اسکریپت را دوباره اجرا کنی.
          </div>
        </div>

        <div class="azy-modal-actions">
          <button
            class="azy-btn"
            id="azy-close-cancel"
            type="button"
          >
            انصراف
          </button>

          <button
            class="azy-btn azy-modal-danger"
            id="azy-close-confirm"
            type="button"
          >
            بله، ببند
          </button>
        </div>
      </div>
    </div>
`;

  document.body.appendChild(
    root
  );

  // ============================================================
  // TABS
  // ============================================================

  function switchTab(name) {
    document
      .querySelectorAll(
        `#${APP_ID} .azy-tab`
      )
      .forEach(button => {

        button.classList.toggle(
          "active",
          button.dataset.tab === name
        );
      });

    document
      .querySelectorAll(
        `#${APP_ID} .azy-view`
      )
      .forEach(view => {

        view.classList.toggle(
          "active",
          view.dataset.view === name
        );
      });

    if (
      name === "export"
    ) {
      $("#azy-preview-text").value =
        buildOutputText();
    }

    if (
      name === "settings"
    ) {
      updateUI();
    }
  }

  // ============================================================
  // TOAST
  // ============================================================

  let toastTimer;

  function inferToastType(message) {
    const value = String(message || "");

    if (/نامعتبر|خطا|انجام نشد|failed|error/i.test(value)) {
      return "error";
    }

    if (/بدون کلید|خالی|خاموش شد|پیدا نشد|هشدار/i.test(value)) {
      return "warning";
    }

    if (/ذخیره شد|کپی شد|ساخته شد|روشن شد|انتخاب شد|پیدا شد|دانلود|✓/i.test(value)) {
      return "success";
    }

    return "info";
  }

  function toast(message, type = "auto", duration = 3200, force = false) {
    const el = $("#azy-toast");

    if (!el) {
      return;
    }

    if (
      !force &&
      state.settings.notifications === false &&
      type !== "error"
    ) {
      return;
    }

    const resolvedType =
      type === "auto"
        ? inferToastType(message)
        : type;

    const config = {
      success: {
        icon: "✓",
        title: "انجام شد"
      },

      warning: {
        icon: "!",
        title: "توجه"
      },

      error: {
        icon: "×",
        title: "خطا"
      },

      info: {
        icon: "i",
        title: "آزمونیار"
      }
    }[resolvedType] || {
      icon: "i",
      title: "آزمونیار"
    };

    el.dataset.type = resolvedType;

    const icon = $("#azy-toast-icon");
    const title = $("#azy-toast-title");
    const body = $("#azy-toast-message");
    const progress = el.querySelector(".azy-toast-progress");

    if (icon) {
      icon.textContent = config.icon;
    }

    if (title) {
      title.textContent = config.title;
    }

    if (body) {
      body.textContent = String(message);
    }

    if (progress) {
      progress.style.setProperty(
        "--azy-toast-duration",
        `${duration}ms`
      );

      progress.style.animation = "none";
      void progress.offsetWidth;
      progress.style.animation = "";
    }

    el.classList.add("show");

    clearTimeout(toastTimer);

    toastTimer =
      setTimeout(
        () => {
          el.classList.remove("show");
        },
        duration
      );
  }

  function hideToast() {
    clearTimeout(toastTimer);
    $("#azy-toast")?.classList.remove("show");
  }

  // ============================================================
  // ONBOARDING
  // ============================================================

  let onboardingStep = 1;
  const ONBOARDING_STEPS = 5;

  function setOnboardingStep(step) {
    onboardingStep =
      Math.min(
        ONBOARDING_STEPS,
        Math.max(1, Number(step) || 1)
      );

    state.onboarding.lastStep =
      onboardingStep;

    saveState();

    document
      .querySelectorAll(
        `#${APP_ID} [data-onboarding-step]`
      )
      .forEach(section => {
        section.classList.toggle(
          "active",
          Number(
            section.dataset.onboardingStep
          ) === onboardingStep
        );
      });

    document
      .querySelectorAll(
        `#${APP_ID} [data-onboarding-dot]`
      )
      .forEach(dot => {
        dot.classList.toggle(
          "active",
          Number(
            dot.dataset.onboardingDot
          ) === onboardingStep
        );
      });

    const count =
      $("#azy-onboarding-count");

    if (count) {
      count.textContent =
        `${onboardingStep} / ${ONBOARDING_STEPS}`;
    }

    const bar =
      $("#azy-onboarding-progress-bar");

    if (bar) {
      bar.style.width =
        `${onboardingStep / ONBOARDING_STEPS * 100}%`;
    }

    const previous =
      $("#azy-onboarding-prev");

    if (previous) {
      previous.disabled =
        onboardingStep === 1;

      previous.style.opacity =
        onboardingStep === 1
          ? ".45"
          : "1";
    }

    const next =
      $("#azy-onboarding-next");

    if (next) {
      next.textContent =
        onboardingStep === ONBOARDING_STEPS
          ? "شروع استفاده"
          : "بعدی";
    }
  }

  function showOnboarding(force = false) {
    const onboarding =
      state.onboarding || DEFAULT_STATE.onboarding;

    if (!force) {
      if (onboarding.disabled) {
        return;
      }

      if (onboarding.completed) {
        return;
      }

      if (
        onboarding.remindAfter &&
        Date.now() < onboarding.remindAfter
      ) {
        return;
      }
    }

    const startStep =
      force
        ? 1
        : onboarding.lastStep || 1;

    setOnboardingStep(startStep);

    $("#azy-onboarding-modal")
      ?.classList.add(
        "show"
      );

    setTimeout(
      () =>
        $("#azy-onboarding-next")
          ?.focus(),
      30
    );
  }

  function hideOnboarding() {
    $("#azy-onboarding-modal")
      ?.classList.remove(
        "show"
      );
  }

  function completeOnboarding() {
    state.onboarding.completed =
      true;

    state.onboarding.disabled =
      false;

    state.onboarding.remindAfter =
      0;

    state.onboarding.lastStep =
      1;

    saveState();
    hideOnboarding();

    $("#azy-panel")
      ?.classList.remove(
        "azy-hidden"
      );

    $("#azy-bubble")
      ?.classList.add(
        "azy-hidden"
      );

    toast(
      "آماده است؛ میتوانی از خانه شروع کنی.",
      "success",
      3200,
      true
    );
  }

  function remindOnboardingLater() {
    state.onboarding.completed =
      false;

    state.onboarding.disabled =
      false;

    state.onboarding.remindAfter =
      Date.now() +
      24 * 60 * 60 * 1000;

    state.onboarding.lastStep =
      onboardingStep;

    saveState();
    hideOnboarding();

    toast(
      "راهنما تا ۲۴ ساعت دیگر نمایش داده نمیشود.",
      "info",
      3000,
      true
    );
  }

  function disableOnboarding() {
    state.onboarding.completed =
      true;

    state.onboarding.disabled =
      true;

    state.onboarding.remindAfter =
      0;

    state.onboarding.lastStep =
      1;

    saveState();
    hideOnboarding();

    toast(
      "نمایش خودکار راهنما غیرفعال شد؛ از Settings میتوانی دوباره آن را باز کنی.",
      "info",
      3800,
      true
    );
  }

  function resetOnboardingForNextRun() {
    state.onboarding.completed =
      false;

    state.onboarding.disabled =
      false;

    state.onboarding.remindAfter =
      0;

    state.onboarding.lastStep =
      1;

    saveState();
    updateUI();

    toast(
      "Onboarding در اجرای بعدی دوباره نمایش داده میشود.",
      "success"
    );
  }

  // ============================================================
  // UPDATE UI
  // ============================================================

  function updateUI() {
    const current =
      getQuestionContainers().length;

    const saved =
      getSortedQuestions().length;

    const expected =
      getExpectedQuestionCount();

    $("#azy-current").textContent =
      current;

    $("#azy-saved").textContent =
      saved;

    $("#azy-expected").textContent =
      expected;

    $("#azy-badge").textContent =
      saved;

    $("#azy-progress-text").textContent =
      `${saved} / ${expected}`;

    const percentage =
      expected
        ? Math.min(
            100,
            saved /
              expected *
              100
          )
        : 0;

    $("#azy-progress-bar").style.width =
      `${percentage}%`;

    $("#azy-auto-capture").checked =
      state.settings.autoCapture;

    $("#azy-auto-fill").checked =
      state.settings.autoFill;

    const settingsAutoCapture =
      $("#azy-settings-auto-capture");

    if (settingsAutoCapture) {
      settingsAutoCapture.checked =
        state.settings.autoCapture;
    }

    const settingsAutoFill =
      $("#azy-settings-auto-fill");

    if (settingsAutoFill) {
      settingsAutoFill.checked =
        state.settings.autoFill;
    }

    const openOnStart =
      $("#azy-open-on-start");

    if (openOnStart) {
      openOnStart.checked =
        !!state.settings.openOnStart;
    }

    const notifications =
      $("#azy-notifications");

    if (notifications) {
      notifications.checked =
        state.settings.notifications !== false;
    }

    const storageSize =
      $("#azy-storage-size");

    if (storageSize) {
      storageSize.textContent =
        formatBytes(
          getStorageSize()
        );
    }

    const examTitle =
      $("#azy-exam-title");

    if (examTitle) {
      examTitle.textContent =
        getExamTitle();
    }

    const statusChip =
      $("#azy-status-chip");

    if (statusChip) {
      if (
        expected > 0 &&
        saved >= expected
      ) {
        statusChip.textContent =
          "کامل";

        statusChip.style.borderColor =
          "rgba(16,185,129,.26)";

        statusChip.style.background =
          "rgba(16,185,129,.09)";

        statusChip.style.color =
          "#6ee7b7";
      } else {
        statusChip.textContent =
          "در حال جمعآوری";

        statusChip.style.borderColor =
          "rgba(99,102,241,.24)";

        statusChip.style.background =
          "rgba(99,102,241,.08)";

        statusChip.style.color =
          "#c7d2fe";
      }
    }
  }

  // ============================================================
  // EVENTS
  // ============================================================

  $("#azy-bubble").onclick =
    () => {

      $("#azy-panel")
        .classList.remove(
          "azy-hidden"
        );

      $("#azy-bubble")
        .classList.add(
          "azy-hidden"
        );
    };

  $("#azy-minimize").onclick =
    () => {

      $("#azy-panel")
        .classList.add(
          "azy-hidden"
        );

      $("#azy-bubble")
        .classList.remove(
          "azy-hidden"
        );
    };

  $("#azy-close").onclick =
    () => {
      $("#azy-close-modal")
        ?.classList.add(
          "show"
        );

      setTimeout(
        () => {
          $("#azy-close-cancel")
            ?.focus();
        },
        20
      );
    };

  $("#azy-close-cancel").onclick =
    () => {
      $("#azy-close-modal")
        ?.classList.remove(
          "show"
        );
    };

  $("#azy-close-confirm").onclick =
    () => {
      $("#azy-close-modal")
        ?.classList.remove(
          "show"
        );

      $("#azy-panel")
        ?.classList.add(
          "azy-hidden"
        );

      $("#azy-bubble")
        ?.classList.add(
          "azy-hidden"
        );

      toast(
        "آزمونیار بسته شد؛ دادههای ذخیرهشده محفوظ هستند.",
        "info",
        2400
      );
    };

  $("#azy-close-modal").addEventListener(
    "click",
    event => {
      if (
        event.target ===
        $("#azy-close-modal")
      ) {
        $("#azy-close-modal")
          .classList.remove(
            "show"
          );
      }
    }
  );

  $("#azy-toast-close").onclick =
    hideToast;

  document.addEventListener(
    "keydown",
    event => {
      if (event.key !== "Escape") {
        return;
      }

      if (
        $("#azy-onboarding-modal")
          ?.classList.contains(
            "show"
          )
      ) {
        remindOnboardingLater();
        return;
      }

      if (
        $("#azy-data-modal")
          ?.classList.contains(
            "show"
          )
      ) {
        $("#azy-data-modal")
          ?.classList.remove(
            "show"
          );

        $("#azy-clear-current-exam")
          ?.focus();

        return;
      }

      if (
        $("#azy-close-modal")
          ?.classList.contains(
            "show"
          )
      ) {
        $("#azy-close-modal")
          .classList.remove(
            "show"
          );

        $("#azy-close")
          ?.focus();
      }
    }
  );

  document
    .querySelectorAll(
      `#${APP_ID} .azy-tab`
    )
    .forEach(button => {

      button.onclick =
        () =>
          switchTab(
            button.dataset.tab
          );
    });

  $("#azy-scan").onclick =
    () =>
      scanAllQuestions(false);

  $("#azy-copy").onclick =
    copyOutput;

  $("#azy-copy-2").onclick =
    copyOutput;

  $("#azy-download").onclick =
    downloadOutput;

  $("#azy-download-2").onclick =
    downloadOutput;

  $("#azy-preview").onclick =
    () =>
      switchTab(
        "export"
      );

  $("#azy-save-key").onclick =
    saveAnswerKey;

  $("#azy-fill").onclick =
    () =>
      fillCurrentAnswers(
        false
      );

  $("#azy-fill-2").onclick =
    () =>
      fillCurrentAnswers(
        false
      );

  $("#azy-check").onclick =
    checkAnswers;

  $("#azy-auto-capture").onchange =
    event => {

      state.settings.autoCapture =
        event.target.checked;

      saveState();
      updateUI();

      toast(
        event.target.checked
          ? "ذخیره خودکار روشن شد"
          : "ذخیره خودکار خاموش شد"
      );
    };

  $("#azy-auto-fill").onchange =
    event => {

      state.settings.autoFill =
        event.target.checked;

      saveState();
      updateUI();

      if (
        event.target.checked
      ) {
        fillCurrentAnswers(
          true
        );
      }

      toast(
        event.target.checked
          ? "Auto Fill روشن شد"
          : "Auto Fill خاموش شد"
      );
    };

  const settingsAutoCapture =
    $("#azy-settings-auto-capture");

  if (settingsAutoCapture) {
    settingsAutoCapture.onchange =
      event => {
        state.settings.autoCapture =
          event.target.checked;

        saveState();
        updateUI();

        toast(
          event.target.checked
            ? "ذخیره خودکار روشن شد"
            : "ذخیره خودکار خاموش شد"
        );
      };
  }

  const settingsAutoFill =
    $("#azy-settings-auto-fill");

  if (settingsAutoFill) {
    settingsAutoFill.onchange =
      event => {
        state.settings.autoFill =
          event.target.checked;

        saveState();
        updateUI();

        if (event.target.checked) {
          fillCurrentAnswers(true);
        }

        toast(
          event.target.checked
            ? "Auto Fill روشن شد"
            : "Auto Fill خاموش شد"
        );
      };
  }

  $("#azy-open-on-start").onchange =
    event => {
      state.settings.openOnStart =
        event.target.checked;

      saveState();
      updateUI();

      toast(
        event.target.checked
          ? "در اجرای بعدی پنل بهصورت باز شروع میشود."
          : "در اجرای بعدی آزمونیار با حباب کوچک شروع میشود.",
        "info"
      );
    };

  $("#azy-notifications").onchange =
    event => {
      state.settings.notifications =
        event.target.checked;

      saveState();
      updateUI();

      toast(
        event.target.checked
          ? "اعلانهای Toast روشن شدند."
          : "اعلانهای Toast خاموش شدند.",
        "info",
        2800,
        true
      );
    };

  $("#azy-show-onboarding").onclick =
    () => {
      showOnboarding(true);
    };

  $("#azy-reset-onboarding").onclick =
    resetOnboardingForNextRun;

  $("#azy-backup-data").onclick =
    downloadBackup;

  $("#azy-clear-current-exam").onclick =
    () => {
      $("#azy-data-modal")
        ?.classList.add(
          "show"
        );

      setTimeout(
        () =>
          $("#azy-data-cancel")
            ?.focus(),
        20
      );
    };

  $("#azy-data-cancel").onclick =
    () => {
      $("#azy-data-modal")
        ?.classList.remove(
          "show"
        );
    };

  $("#azy-data-confirm").onclick =
    () => {
      $("#azy-data-modal")
        ?.classList.remove(
          "show"
        );

      clearCurrentExamData();
    };

  $("#azy-data-modal").addEventListener(
    "click",
    event => {
      if (
        event.target ===
        $("#azy-data-modal")
      ) {
        $("#azy-data-modal")
          ?.classList.remove(
            "show"
          );
      }
    }
  );

  $("#azy-onboarding-prev").onclick =
    () => {
      setOnboardingStep(
        onboardingStep - 1
      );
    };

  $("#azy-onboarding-next").onclick =
    () => {
      if (
        onboardingStep >=
        ONBOARDING_STEPS
      ) {
        completeOnboarding();
        return;
      }

      setOnboardingStep(
        onboardingStep + 1
      );
    };

  $("#azy-onboarding-later").onclick =
    remindOnboardingLater;

  $("#azy-onboarding-never").onclick =
    disableOnboarding;

  // ============================================================
  // WATCH DOM
  // ============================================================

  let mutationTimer;

  const observer =
    new MutationObserver(
      mutations => {

        // Ignore our own UI mutations

        const relevant =
          mutations.some(
            mutation => {

              const target =
                mutation.target;

              return !(
                target instanceof Element &&
                target.closest(
                  `#${APP_ID}`
                )
              );
            }
          );

        if (!relevant) {
          return;
        }

        clearTimeout(
          mutationTimer
        );

        mutationTimer =
          setTimeout(
            () => {

              if (
                state.settings
                  .autoCapture
              ) {
                scanAllQuestions(
                  true
                );
              }

              if (
                state.settings
                  .autoFill
              ) {
                fillCurrentAnswers(
                  true
                );
              }

              updateUI();

            },
            400
          );
      }
    );

  observer.observe(
    document.body,
    {
      childList: true,
      subtree: true
    }
  );

  // ============================================================
  // BEFORE NAVIGATION CAPTURE
  // ============================================================

  document.addEventListener(
    "click",
    event => {

      if (
        event.target.closest(
          "#mod_quiz-next-nav,.mod_quiz-next-nav,.qnbutton"
        )
      ) {
        scanAllQuestions(
          true
        );
      }
    },
    true
  );

  // ============================================================
  // INITIAL SCAN
  // ============================================================

  scanAllQuestions(
    true
  );

  updateUI();

  if (
    state.settings.openOnStart
  ) {
    $("#azy-panel")
      ?.classList.remove(
        "azy-hidden"
      );

    $("#azy-bubble")
      ?.classList.add(
        "azy-hidden"
      );
  }

  setTimeout(
    () => {
      showOnboarding(false);
    },
    550
  );

  console.log(
    `%cآزمونیار v3.2 فعال شد — ${getSortedQuestions().length}/${getExpectedQuestionCount()} سؤال پیدا شد`,
    `
      color:#10b981;
      font-size:15px;
      font-weight:bold
    `
  );
})();
