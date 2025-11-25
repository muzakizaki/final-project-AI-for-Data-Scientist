# Cell 1: Memuat Data dan Pembersihan Awal (VERSI FINAL yang dijamin aman)

import pandas as pd
# ... (import lainnya)

# Daftar semua file CSV
files = ['clips.csv', 'gamesession.csv', 'downloaded_clips.csv',
         'shared_clips.csv', 'premium.csv']

df_dict = {}

for file in files:
    try:
        df_name = file.replace('.csv', '')
        df_dict[df_name] = pd.read_csv(file)

        # PERBAIKAN 1: Hapus spasi dari semua nama kolom
        df_dict[df_name].columns = df_dict[df_name].columns.str.strip()

        # PERBAIKAN 2: Ganti nama kolom yang diketahui menggunakan user_Id (I besar)
        if 'user_Id' in df_dict[df_name].columns:
            # Ganti user_Id (I besar) menjadi user_id (i kecil)
            df_dict[df_name] = df_dict[df_name].rename(columns={'user_Id': 'user_id'})

        # Perbaikan Khusus untuk gamesession jika ada masalah lain:
        if 'user_id' not in df_dict[df_name].columns and 'user_id ' in df_dict[df_name].columns:
             df_dict[df_name] = df_dict[df_name].rename(columns={'user_id ': 'user_id'})

        print(f"Loaded {df_name} with {len(df_dict[df_name])} rows.")
    except FileNotFoundError:
        print(f"Error: File {file} not found.")

# FUNGSI KONVERSI WAKTU (JANGAN DIUBAH)
def convert_to_datetime(df, cols):
    for col in cols:
        if col in df.columns:
            # Mengkonversi dari timestamp (detik) ke datetime
            df[col] = pd.to_datetime(df[col], unit='s', errors='coerce')
    return df

# Konversi semua kolom timestamp di setiap tabel
df_dict['gamesession'] = convert_to_datetime(df_dict['gamesession'], ['submited_date', 'created_at', 'join_at'])
df_dict['clips'] = convert_to_datetime(df_dict['clips'], ['created_at', 'join_at'])
df_dict['downloaded_clips'] = convert_to_datetime(df_dict['downloaded_clips'], ['created_at', 'join_at'])
df_dict['shared_clips'] = convert_to_datetime(df_dict['shared_clips'], ['created_at', 'scheduled_at', 'join_at'])
df_dict['premium'] = convert_to_datetime(df_dict['premium'], ['starts_at', 'ends_at', 'created_at', 'updated_at', 'canceled_at', 'deleted_at', 'join_at'])

print("\nKonversi waktu selesai. Data siap dianalisis.")


# --- 2. KONFIGURASI DAN INISIALISASI ---
MODEL_NAME = 'gemini-2.5-flash'
INPUT_CSV = "Game Thumbnail.csv"
OUTPUT_CSV = "Game_Data_Enhanced_Final.csv"
MAX_RETRIES = 3 # Jumlah percobaan ulang jika API gagal

try:
    # Mengambil Kunci API secara aman dari lingkungan
    api_key = os.getenv("GEMINI_API_KEY")
    if not api_key:
        # Ini penting untuk skrip lokal. Di Colab, Anda bisa menggunakan userdata.get()
        raise ValueError("GEMINI_API_KEY tidak ditemukan. Harap atur variabel lingkungan Anda.")

    client = genai.Client(api_key=api_key)
    print("Klien Gemini berhasil diinisialisasi.")
except Exception as e:
    print(f"ERROR INISIALISASI: {e}")
    exit(1)


    # --- 3. PROMPT ENGINEERING FUNCTION (CORE LOGIC) ---

def generate_content_with_retry(prompt, game_title, max_retries=MAX_RETRIES):
    """
    Fungsi untuk memanggil API dengan mekanisme percobaan ulang sederhana (Error Handling).
    Jika terjadi APIError, ia akan menunggu sebentar (backoff) dan mencoba lagi.
    """
    # Menggabungkan judul game ke dalam prompt untuk konteks yang jelas
    full_prompt = f"Game Title: '{game_title}'. {prompt}"

    for attempt in range(max_retries):
        try:
            response = client.models.generate_content(
                model=MODEL_NAME,
                contents=full_prompt,
                config=genai.types.GenerateContentConfig(
                    temperature=0.0  # Suhu 0.0 memaksa konsistensi dan kepatuhan format
                )
            )
            # Membersihkan output dari spasi berlebih atau kutipan yang tidak perlu
            return response.text.strip().replace('"', '').replace("'", "")

        except APIError as e:
            print(f"  ⚠️ API Gagal untuk '{game_title}' (Percobaan {attempt + 1}). Error: {e}")
            if attempt < max_retries - 1:
                # Menunggu secara eksponensial (1s, 2s, 4s) sebelum mencoba lagi
                time.sleep(2 ** attempt)
            else:
                return "Unknown/Error"
        except Exception as e:
            # Menangkap error non-API lainnya
            print(f"  ❌ Error tidak terduga: {e}")
            return "Unknown/Error"
    return "Unknown/Error"

# Definisi prompt presisi untuk setiap kolom (Membantu klasifikasi AI)
GENRE_PROMPT = "Klasifikasikan genre utamanya sebagai SATU KATA. Jawab HANYA dengan kata genre. Contoh: 'Action'."
DESCRIPTION_PROMPT = "Buat deskripsi singkat dan menarik untuk game ini. Deskripsi harus kurang dari 30 kata. Jawab HANYA dengan deskripsi singkat."
PLAYER_MODE_PROMPT = "Tentukan mode pemain utama. Jawab HANYA dengan salah satu dari tiga kata ini: 'Singleplayer', 'Multiplayer', atau 'Both'."



# --- 4. ALUR KERJA UTAMA (EKSEKUSI) ---

if __name__ == "__main__":
    try:
        df = pd.read_csv(INPUT_CSV)
    except FileNotFoundError:
        print(f"❌ ERROR: File '{INPUT_CSV}' tidak ditemukan.")
        exit(1)

    print(f"Data dimuat. Total {len(df)} entri untuk diproses.")

    # Inisialisasi kolom baru
    df['genre'] = ""
    df['short_description'] = ""
    df['player_mode'] = ""

    print("\n--- Memulai Pemrosesan Data dengan Gemini AI ---")
    for index, row in df.iterrows():
        title = row['game_title']

        # 1. Genre
        df.loc[index, 'genre'] = get_genre(title, GENRE_PROMPT)

        # 2. Short Description
        df.loc[index, 'short_description'] = get_description(title, DESCRIPTION_PROMPT)

        # 3. Player Mode
        df.loc[index, 'player_mode'] = get_player_mode(title, PLAYER_MODE_PROMPT)

        # Menampilkan progress
        print(f"[{index + 1}/{len(df)}] Proses: {title}")

    # Simpan Enhanced CSV
    df.to_csv(OUTPUT_CSV, index=False)
    print(f"\n✅ Semua entri selesai. Data disimpan ke: {OUTPUT_CSV}")
