# תמונת בסיס רשמית של Nginx בגרסה קבועה (1.27), על Alpine: קטנה ומהירה להורדה
FROM nginx:1.27-alpine

# מחליף את קובץ ההגדרות של ברירת המחדל בהגדרות שלנו (אתר סטטי, /health, פרוקסי ל-/plates/)
COPY nginx.conf /etc/nginx/conf.d/default.conf
# מעתיק רק את הדף (HTML+CSS+JS בקובץ אחד) לתיקייה ש-Nginx מגיש ממנה; לא את כל התיקייה, כדי לא לחשוף את nginx.conf
COPY index.html /usr/share/nginx/html/

# תיעוד: הקונטיינר מאזין בפורט 80 (הפתיחה בפועל היא ב-ports ב-docker-compose.yml)
EXPOSE 80
