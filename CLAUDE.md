# Xonix 3D — הנחיות לעבודה על הפרויקט

משחק Xonix תלת-מימדי, קובץ `index.html` יחיד. **האילוץ המרכזי: קובץ יחיד, בלי build step.** זו החלטה מכוונת של המשתמש (לא מגבלה טכנית) - אל תציע webpack/vite/bundler או פיצול לקבצים מרובים בלי לשאול קודם.

## גרסת Three.js ו-ES Modules

טעון דרך CDN (jsdelivr) כ-**ES modules** עם import map (`<script type="importmap">` + `<script type="module">`), לא כ-UMD global. זה נדרש כדי לשתף בדיוק את אותו מופע Three.js בין המשחק לבין תוספי הפוסט-פרוססינג (`EffectComposer`/`RenderPass`/`UnrealBloomPass`/`ShaderPass`/`RoomEnvironment` מ-`three/addons/`) — ערבוב UMD global עם ES module נפרד של אותה גרסה יוצר שתי מחלקות `THREE.*` שונות ועלול לשבור בדיקות `instanceof` פנימיות בתוספים.

**נעול על `three@0.149.0`** — זו הגרסה האחרונה שעדיין מפרסמת גם build UMD ממוזער (למקרה שנחזור אליו) וגם `examples/jsm` תואם. אל תשדרג בלי לבדוק זמינות בשתי הצורות. (`OutputPass` לא קיים ב-0.149 - 404.)

פונטים (Rubik + Orbitron) נטענים מ-Google Fonts ב-`<head>`. אין ספריות JS נוספות מעבר ל-three.

## צינור צבע ורינדור (אחרי שדרוג הגרפיקה)

- **`THREE.ColorManagement.legacyMode = false`** מוגדר מיד אחרי ה-imports - כל `new THREE.Color` שנוצר לפני השורה הזו לא יומר מ-sRGB לליניארי.
- הסצנה מרונדרת ל-render target **HalfFloat + MSAA** בצבע ליניארי. פאס ה-composite האחרון עושה: base + bloom → ACES → vignette/chromatic aberration → **המרה ידנית ל-sRGB** → dither.
  - **אל תגדירו `renderer.toneMapping`** - ב-r149 הוא מופעל גם בתוך render targets, כלומר לפני הוספת ה-bloom.
  - **אל תסירו את `toSRGB` מה-composite** - `ShaderMaterial` מותאם לא קורא `linearToOutputTexel`, ו-`renderer.outputEncoding` מתעלם ממנו.
  - צבעי זוהר HDR נוצרים עם `Color.multiplyScalar(>1)`. כוונון בהירות כללית: `compositePass.uniforms.uExposure`.
- אנטי-אליאסינג מגיע מ-MSAA של ה-render target (ה-`antialias` של הקנבס חסר תועלת מאחורי EffectComposer). **pixel ratio מוגבל ל-`Render.MAX_PIXEL_RATIO` (1.5)**, לא 2 - העלות תלויה בכמות הפיקסלים.
- **מראה הרשת (קווי ניאון על גבולות קירות, צל מגע ברצפה, פעימת שביל) חי ב-`onBeforeCompile` של `Render.initGrid`** וקורא DataTexture של מצבי התאים (`Render.gridTex`/`gridTexData`). כל שינוי ב-`Grid.state` חייב לעבור דרך `Grid.setState` → dirty → `flushDirty`, אחרת הקווים לא יתעדכנו.
- `Render.playerMesh` ו-`Render.enemyMeshes[i]` הם **Group** (אין להם `.material`). חומרי הכדורים **משותפים** - צביעת כדור אחד צובעת את כולם.

## Gotchas ברינדור

- **`InstancedMesh` + `vertexColors` דורש attribute `color` בסיסי בגאומטריה** (לבן, לכל vertex) - בלי זה `vColor` מתחיל כשחור (0,0,0) וצבעי ה-instance נראים שחורים לגמרי למרות שהערכים נכונים. תמיד להוסיף:
  ```js
  geo.setAttribute('color', new THREE.BufferAttribute(new Float32Array(vertCount*3).fill(1), 3));
  ```
- **`RoundedBoxGeometry` לתאי הרשת נראה טוב בתיאוריה אך יוצר תפר דק בכל גבול תא** (נורמלים שונים על שיפוע ה-bevel) - סותר את דרישת "משטח חלק ללא חלוקה לריבועים". נבדק ונדחה - השתמשו ב-`BoxGeometry` פשוט. מאותה סיבה אין קווי רשת פר-תא ברצפה (רק רשת עדינה כל 4 תאים).
- **מצלמת ה-frustum מחושבת דינמית** (`Render.fitCameraToBoard`) לפי יחס הרוחב-גובה הנוכחי, לא ערך קבוע - ערך קבוע נשבר בגדלי חלון שונים (הלוח חורג מהתצוגה). הפונקציה שומרת את התנוחה ב-`Render.cameraBasePos`, ו-`FX.applyCamera` מוסיף מעליה בכל פריים את פתיח השלב ורעידת המצלמה. **אל תקבעו `camera.position` מחוץ לשני אלה.**
- **Bloom הוא Selective (layer-based), לא threshold-based.** רק אובייקט עם `layers.enable(1)` זוהר (חלקי החללית, כדורים, פאוור-אפים, חלקיקים, טבעות, מגן) - הרשת (`InstancedMesh` של קירות/שביל/רצפה) אף פעם לא זוהרת, ללא קשר לצבעה. אם מוסיפים אובייקט חדש שאמור לזהור, חובה `layers.enable(1)` עליו, אחרת בפאס ה-bloom (`_darkenNonBloomed`) Mesh ייצבע שחור ו-Points/Lines/Sprites יוסתרו. Mesh שקוף שלא בשכבה 1 יצויר כשחור אטום ויסתיר זוהר שמאחוריו - שימו אותו בשכבה 1.
- **`Grid.init(level)` מקבל פרמטר `level` (לא ריק!)** - מ-שלב 3 ואילך זה זורע מכשולי קיר קבועים (`Grid.addLevelObstacles`). קריאה בלי `level` (או `level<3`) מדלגת על מכשולים - זה מכוון (שלבי פתיחה נשארים פשוטים), לא באג. ה-PRNG למכשולים (mulberry32) דטרמיניסטי עם seed=`level*7919+13`, אז אותו שלב תמיד מקבל אותה פריסה - שימושי לבדיקות חוזרות.
- **סדר קריאות קריטי ב-`beginNewGame()`: `Player.init()` לפני `Grid.init(level)`.** מכשולי השלב נמנעים מאזור התחלת השחקן (`Player.startCol/startRow`) - אם הסדר יתהפך, `startCol/startRow` עדיין 0 בזמן זריעת המכשולים.

## מודול FX (תנועה ומשוב - ויזואלי בלבד)

- `const FX` לא משנה מצב משחק אף פעם. **גובה התאים כבר לא מתעדכן מיידית**: `Render.flushDirty` מעביר את התאים המלוכלכים ל-`FX.cells.applyDirty`, שמנפיש אותם עד ~0.85 שנ׳. בדיקה שקוראת `instanceMatrix` מיד אחרי כיבוש צריכה לקדם גם את `FX.update`/`FX.lateUpdate`, או לקרוא את `Grid.state`.
- ב-`animate()`: `fxDt` (0 כשמושהה) משמש לכל טיימר ויזואלי. `gdt = dt * FX.timeScale` משמש לעדכוני המשחקיות (רגע קפאון/הילוך איטי במוות). טיימר משחקיות חדש - `gdt` בתוך בלוק ה-PLAYING. טיימר ויזואלי חדש - `fxDt`.
- סדר חובה בלולאה: `FX.lateUpdate` רץ **אחרי** `updatePlayerMesh`/`updateEnemyMeshes` (דורס סיבוב/סקייל) ו**לפני** `updateParticles`/`render`.
- תזמון סיום שלב: הבאנר החגיגי נראה על הלוח, מסך "שלב הושלם" מופיע אחרי 1.5 שנ׳, והשלב הבא מתחיל אחרי 3.0 שנ׳. אנימציית הפס ב-CSS (`lcBar 1.5s`) מסונכרנת לזה. הבונוס מחושב גם ב-`UI.showLevelComplete` - שינוי בנוסחה ב-`Game.onLevelComplete` מחייב עדכון גם שם.
- פאוור-אפ בתוך שטח שנכבש נאסף אוטומטית כבונוס (אחרת נקבר בתוך הקיר).

## ממשק (DOM)

- **מסכים (`.screen`) לא נעלמים עם `display:none`** אלא עם opacity+visibility, כדי שיהיו מעברים. `classList.add/remove('hidden')` עובד כרגיל. מסך חדש צריך רק `class="screen hidden"`. על כל אלמנט אחר `.hidden` עדיין אומר `display:none`.
- ה-SVG sprite (`#i-heart` וכו') חייב להישאר ב-`<body>` לפני ה-HUD - הלבבות נבנים עם `innerHTML` ומפנים אליו.
- צ'יפים של פאוור-אפים ב-HUD נוצרים פעם אחת ורק מתעדכנים - אל תחזירו `innerHTML` בכל פריים.

## מצב מושהה

- **`Game.state === 'PAUSED'` מקפיא הכל "בחינם"** כי כל עדכוני המשחקיות (`Player.update`/`Enemies.update`/`PowerUps.update`) כבר מגוננים מאחורי `if (Game.state === 'PLAYING')` ב-`animate()`, ו-`fxDt` הוא 0. אם מוסיפים מנגנון חדש עם טיימר משלו (כמו power-up), חובה לוודא שהעדכון שלו קורה רק בתוך אותו בלוק מוגן, אחרת הוא ימשיך לרוץ גם בהשהיה. (אנימציות שיידר קוסמטיות - פעימת שביל, שמיים - ממשיכות לזוז בהשהיה, בכוונה.)

## סאונד - הוסר במכוון, אל תוסיפו בלי לשאול

נוסה מודול `SFX` מלא (Web Audio API, אוסילטורים סינתטיים - tick/whoosh/buzz/chime/פנפייר + כפתור השתקה), **הוטמע ונדחף (קומיט `93b1413`), ואז הוסר לגמרי (קומיט `2c2ed47`) לפי בקשה מפורשת של המשתמש** ("לא אהבתי את הסאונד, תבטל אותו לגמרי"). זו לא בעיה טכנית - הסאונד עבד תקין. **אל תוסיפו סאונד מחדש בלי לשאול את המשתמש קודם**, גם אם מתבקש "שפר את חווית המשחק" באופן כללי.

## בדיקה בדפדפן

- הרחבת Claude-in-Chrome **חוסמת ניווט ל-`file://`**. יש להריץ שרת סטטי מקומי (`python -m http.server`) ולנווט ל-`http://127.0.0.1:PORT/index.html`. הוסיפו `?v=N` ל-URL אחרי כל שינוי - אחרת הדפדפן מגיש גרסה מה-cache.
- מכיוון שהקוד הוא ES module, `const` ברמה עליונה (Game, Render, Player, FX וכו') **לא נחשפים אוטומטית ל-`window`**. לצורך דיבאג בדפדפן אפשר להוסיף זמנית בסוף הקובץ `window.__debug = { Game, Render, Player, ... };` - **אך יש להסיר לפני commit**, זה לא חלק מהקוד המיוצר.
- טאבים ברקע (headless-ish automation) לפעמים מקפיאים/מאטים מאוד את `requestAnimationFrame` - אל תסתמכו על `wait()` בזמן אמת לבדיקת תנועה; העדיפו לולאה עם `dt` קבוע שקוראת `FX.update` → `Player.update`/`Enemies.update`/`Game.checkDeaths` → `Render.flushDirty` → `FX.lateUpdate`. שימו לב שלולאת ה-rAF האמיתית ממשיכה לרוץ במקביל לבדיקה.
- השחקן לא יכול להתהפך על השביל של עצמו (`canMoveTo` חוסם `TRAIL`) - בדיקה שמנסה "לחזור אחורה" לקיר לא תסגור לולאה.

## פריסה

- GitHub Pages מוגש ישירות מ-`master` (root), ללא build. כל push מתעדכן אוטומטית תוך דקה-שתיים.
- ה-CDN וה-Pages שומרים cache (~10 דקות) - אחרי אימות שינוי מול הריפו, אם המשתמש "לא רואה" את השינוי זה כמעט תמיד cache בדפדפן שלו, לא כשל בפריסה. וודאו קודם מול `curl`/diff מול הקובץ המקומי לפני שמניחים שהפריסה נכשלה.
- ריפו: https://github.com/doryamin12/xonix-3d · אתר חי: https://doryamin12.github.io/xonix-3d/

## תיעוד

תהליך הפיתוח המלא, כולל כל הבאגים שנמצאו ותוקנו, מתועד בקובץ Obsidian "Xonix 3D - תיעוד פרויקט" (לא בריפו הזה). עדכנו אותו בכל שינוי משמעותי. בלוקי "קוד מלא" ישנים באותו קובץ אינם מעודכנים - הקוד האמיתי הוא `index.html` בריפו.
