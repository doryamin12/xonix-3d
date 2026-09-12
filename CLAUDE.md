# Xonix 3D — הנחיות לעבודה על הפרויקט

משחק Xonix תלת-מימדי, קובץ `index.html` יחיד. **האילוץ המרכזי: קובץ יחיד, בלי build step.** זו החלטה מכוונת של המשתמש (לא מגבלה טכנית) - אל תציע webpack/vite/bundler או פיצול לקבצים מרובים בלי לשאול קודם.

## גרסת Three.js ו-ES Modules

טעון דרך CDN (jsdelivr) כ-**ES modules** עם import map (`<script type="importmap">` + `<script type="module">`), לא כ-UMD global. זה נדרש כדי לשתף בדיוק את אותו מופע Three.js בין המשחק לבין תוספי הפוסט-פרוססינג (`EffectComposer`/`RenderPass`/`UnrealBloomPass` מ-`three/addons/`) — ערבוב UMD global עם ES module נפרד של אותה גרסה יוצר שתי מחלקות `THREE.*` שונות ועלול לשבור בדיקות `instanceof` פנימיות בתוספים.

**נעול על `three@0.149.0`** — זו הגרסה האחרונה שעדיין מפרסמת גם build UMD ממוזער (למקרה שנחזור אליו) וגם `examples/jsm` תואם. אל תשדרג בלי לבדוק זמינות בשתי הצורות.

## גוchas ברינדור

- **`InstancedMesh` + `vertexColors` דורש attribute `color` בסיסי בגאומטריה** (לבן, לכל vertex) - בלי זה `vColor` מתחיל כשחור (0,0,0) וצבעי ה-instance נראים שחורים לגמרי למרות שהערכים נכונים. תמיד להוסיף:
  ```js
  geo.setAttribute('color', new THREE.BufferAttribute(new Float32Array(vertCount*3).fill(1), 3));
  ```
- **`RoundedBoxGeometry` לתאי הרשת נראה טוב בתיאוריה אך יוצר תפר דק בכל גבול תא** (נורמלים שונים על שיפוע ה-bevel) - סותר את דרישת "משטח חלק ללא חלוקה לריבועים". נבדק ונדחה - השתמשו ב-`BoxGeometry` פשוט.
- **מצלמת ה-frustum מחושבת דינמית** (`Render.fitCameraToBoard`) לפי יחס הרוחב-גובה הנוכחי, לא ערך קבוע - ערך קבוע נשבר בגדלי חלון שונים (הלוח חורג מהתצוגה). כל שינוי במיקום המצלמה חייב לעבור דרך הפונקציה הזו ולהיקרא גם מ-`onResize`.
- **Bloom מכויל ל-threshold=0.6** כך שרק אלמנטים בהירים (חללית/כדורים/פאוור-אפים, luminance~0.75-0.94) זוהרים בעוד הרצפה/קירות הכהים (luminance~0.39) לא. אם משנים צבעי חומרים, בדקו luminance בפועל (`gl.readPixels`) לפני שינוי הסף.

## בדיקה בדפדפן

- הרחבת Claude-in-Chrome **חוסמת ניווט ל-`file://`**. יש להריץ שרת סטטי מקומי (`python -m http.server`) ולנווט ל-`http://127.0.0.1:PORT/index.html`.
- מכיוון שהקוד הוא ES module, `const` ברמה עליונה (Game, Render, Player וכו') **לא נחשפים אוטומטית ל-`window`**. לצורך דיבאג בדפדפן אפשר להוסיף זמנית בסוף הקובץ `window.__debug = { Game, Render, Player, ... };` - **אך יש להסיר לפני commit**, זה לא חלק מהקוד המיוצר.
- טאבים ברקע (headless-ish automation) לפעמים מקפיאים/מאטים מאוד את `requestAnimationFrame` - אל תסתמכו על `wait()` בזמן אמת לבדיקת תנועה; העדיפו קריאה ישירה ל-`Player.update(dt)`/`Enemies.update(dt)` בלולאה עם `dt` קבוע לבדיקה דטרמיניסטית.

## פריסה

- GitHub Pages מוגש ישירות מ-`master` (root), ללא build. כל push מתעדכן אוטומטית תוך דקה-שתיים.
- ה-CDN וה-Pages שומרים cache (~10 דקות) - אחרי אימות שינוי מול הריפו, אם המשתמש "לא רואה" את השינוי זה כמעט תמיד cache בדפדפן שלו, לא כשל בפריסה. וודאו קודם מול `curl`/diff מול הקובץ המקומי לפני שמניחים שהפריסה נכשלה.
- ריפו: https://github.com/doryamin12/xonix-3d · אתר חי: https://doryamin12.github.io/xonix-3d/

## תיעוד

תהליך הפיתוח המלא, כולל כל הבאגים שנמצאו ותוקנו, מתועד בקובץ Obsidian "Xonix 3D - תיעוד פרויקט" (לא בריפו הזה). עדכנו אותו בכל שינוי משמעותי.
