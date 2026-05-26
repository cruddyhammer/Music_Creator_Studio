Manual Steps (For Francisco Only, agent ignore):



**GitHub**

1. Create a new GitHub account (separate from your MRE account)

2. Log into that new account and create a new repository named `MCS` (or whatever you want to call it)

3. Enable GitHub Pages on that repo (Settings → Pages → source: main branch, /docs folder or root)



**Google Drive API — for each of your 3 Google accounts:**

4. Go to Google Cloud Console, create a project, enable the Google Drive API, and create a Service Account with a JSON key downloaded

5. Share a Drive folder with that service account's email so it has write access

6. Repeat steps 4–5 for account 2

7. Repeat steps 4–5 for account 3



**GitHub Actions Secrets — in your new MCS repo:**

8. Add secret: `DRIVE_CREDENTIALS_1` — paste the full JSON content of service account key 1

9. Add secret: `DRIVE_CREDENTIALS_2` — paste the full JSON content of service account key 2

10. Add secret: `DRIVE_CREDENTIALS_3` — paste the full JSON content of service account key 3

11. Add secret: `DRIVE_FOLDER_ID_1` — the folder ID for account 1's target Drive folder

12. Add secret: `DRIVE_FOLDER_ID_2` — folder ID for account 2

13. Add secret: `DRIVE_FOLDER_ID_3` — folder ID for account 3



--- 