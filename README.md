## Praktikum 7-10 

| Atribut         | Keterangan            |
| --------------- | --------------------- |
| **Nama**        | Bayu Aji Yuwono       |
| **NIM**         | 312310492             |
| **Kelas**       | TI.23.A.5             |
| **Mata Kuliah** | Pemrograman Website 2 |

## Membuat Model: `KategoriModel`

membuat model untuk mengelola data kategori melalui CodeIgniter 4.
![{D36B6E3C-61BF-4B76-9FC5-301571EC117A}](https://github.com/user-attachments/assets/1559cb93-995e-400a-9884-0ec58468c3ec)

##  Memodifikasi Model: `ArticleModel`

Setelah membuat `KategoriModel`, langkah selanjutnya adalah memodifikasi model `ArticleModel` agar mendukung relasi dengan tabel `kategori`.
![{3FB62D8F-6E8A-4F08-8DD8-E1EA6109E2A7}](https://github.com/user-attachments/assets/f083671f-1a3b-432f-878d-1af2f3745760)
![{09A51BEA-C674-4D77-9390-FDC374DABC67}](https://github.com/user-attachments/assets/e2ffa945-8249-401c-8919-7afe094678ac)

##  Memodifikasi Controller: `Article.php`
Modifikasi `Artikel.php` untuk menggunakan model baru dan menampilkan data relasi:
```php
<?php

namespace App\Controllers;

use App\Models\ArtikelModel;
use App\Models\KategoriModel;

class Artikel extends BaseController
{
    
    public function index()
    {
        $title = 'Daftar Artikel';
        $model = new ArtikelModel();

        $artikel = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left')
            ->where('artikel.status', 1) // Hanya tampilkan artikel yang aktif
            ->orderBy('artikel.id', 'DESC') // Urutkan dari yang terbaru
            ->findAll();

        return view('artikel/index', compact('artikel', 'title'));
    }

    public function admin_index()
    {
        $title = 'Daftar Artikel';
        $model = new ArtikelModel();
        $kategoriModel = new KategoriModel();

        // Ambil keyword pencarian & filter kategori
        $q = $this->request->getGet('q') ?? '';
        $kategori_id = $this->request->getGet('kategori_id') ?? '';

        // Bangun query
        $builder = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left');

        if (!empty($q)) {
            $builder->like('artikel.judul', $q);
        }

        if (!empty($kategori_id)) {
            $builder->where('artikel.id_kategori', $kategori_id);
        }

        $data = [
            'title' => $title,
            'artikel' => $builder->paginate(10),
            'pager' => $model->pager,
            'q' => $q,
            'kategori_id' => $kategori_id,
            'kategori' => $kategoriModel->findAll(),
        ];

        return view('artikel/admin_index', $data);
    }

    public function add()
    {
        $validation = \Config\Services::validation();

        $validation->setRules([
            'judul' => 'required',
            'isi' => 'required',
            'id_kategori' => 'required|integer',
            'gambar' => [
                'label' => 'Gambar',
                'rules' => 'uploaded[gambar]|is_image[gambar]|max_size[gambar,2048]',
                'errors' => [
                    'uploaded' => 'Gambar wajib diunggah.',
                    'is_image' => 'File harus berupa gambar.',
                    'max_size' => 'Ukuran gambar maksimal 2MB.',
                ],
            ],
        ]);

        if ($validation->withRequest($this->request)->run()) {
            $file = $this->request->getFile('gambar');
            $fileName = $file->getRandomName();
            $file->move(ROOTPATH . 'public/gambar', $fileName);

            $model = new ArtikelModel();
            $model->insert([
                'judul' => $this->request->getPost('judul'),
                'isi' => $this->request->getPost('isi'),
                'slug' => url_title($this->request->getPost('judul'), '-', true),
                'gambar' => $fileName,
                'status' => 1, // Ubah dari 0 (draft) ke 1 (public)
                'id_kategori' => $this->request->getPost('id_kategori'),
            ]);

            return redirect()->to(base_url('admin/artikel'));
        }

        $kategoriModel = new KategoriModel();

        // Cek apakah tabel kategori ada dan memiliki data
        try {
            $kategori = $kategoriModel->findAll();
            if (empty($kategori)) {
                // Jika tidak ada kategori, buat kategori default
                $kategoriModel->insertBatch([
                    ['nama_kategori' => 'Teknologi', 'slug_kategori' => 'teknologi'],
                    ['nama_kategori' => 'Olahraga', 'slug_kategori' => 'olahraga'],
                    ['nama_kategori' => 'Politik', 'slug_kategori' => 'politik'],
                    ['nama_kategori' => 'Ekonomi', 'slug_kategori' => 'ekonomi'],
                    ['nama_kategori' => 'Hiburan', 'slug_kategori' => 'hiburan'],
                ]);
                $kategori = $kategoriModel->findAll();
            }
        } catch (\Exception $e) {
            // Jika tabel kategori tidak ada, buat kategori default
            $kategori = [
                ['id_kategori' => 1, 'nama_kategori' => 'Umum', 'slug_kategori' => 'umum']
            ];
        }

        return view('artikel/form_add', [
            'title' => 'Tambah Artikel',
            'validation' => $validation,
            'kategori' => $kategori
        ]);
    }

    public function edit($id)
    {
        $model = new ArtikelModel();
        $validation = \Config\Services::validation();

        $validation->setRules([
            'judul' => 'required',
            'isi' => 'required',
            'id_kategori' => 'required|integer',
        ]);

        if ($validation->withRequest($this->request)->run()) {
            $model->update($id, [
                'judul' => $this->request->getPost('judul'),
                'isi' => $this->request->getPost('isi'),
                'id_kategori' => $this->request->getPost('id_kategori'),
            ]);

            return redirect()->to('admin/artikel');
        }

        $data = $model->find($id);
        if (!$data) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException("Artikel dengan ID $id tidak ditemukan.");
        }

        $kategoriModel = new KategoriModel();

        // Cek apakah tabel kategori ada dan memiliki data
        try {
            $kategori = $kategoriModel->findAll();
            if (empty($kategori)) {
                $kategori = [
                    ['id_kategori' => 1, 'nama_kategori' => 'Umum', 'slug_kategori' => 'umum']
                ];
            }
        } catch (\Exception $e) {
            $kategori = [
                ['id_kategori' => 1, 'nama_kategori' => 'Umum', 'slug_kategori' => 'umum']
            ];
        }

        return view('artikel/form_edit', [
            'title' => 'Edit Artikel',
            'data' => $data,
            'kategori' => $kategori,
            'validation' => $validation
        ]);
    }

    public function delete($id)
    {
        $model = new ArtikelModel();
        $model->delete($id);

        return redirect()->to('admin/artikel');
    }

    public function view($slug)
    {
        $model = new ArtikelModel();
        $artikel = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left')
            ->where('slug', $slug)
            ->where('artikel.status', 1) // Hanya tampilkan artikel yang aktif
            ->first();

        if (!$artikel) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException("Artikel dengan slug '$slug' tidak ditemukan atau belum dipublikasikan.");
        }

        return view('artikel/view', [
            'title' => $artikel['judul'],
            'artikel' => $artikel
        ]);
    }
}
```

## Memodifikasi View
Buka folder view/artikel sesuaikan masing-masing view.
- index.php
![{417A4590-0293-42DD-9C99-33CB3BF73483}](https://github.com/user-attachments/assets/9eb6e44d-4de3-4c2e-902c-d0a0da4ffe4c)

admin_index.php
```php
<?= $this->include('template/admin_header'); ?>

<h2 class="mt-4 mb-4"><?= esc($title); ?></h2>

<!-- Form Pencarian dan Filter -->
<form method="get" class="row g-2 mb-4" role="search">
    <div class="col-md-4">
        <input type="text" name="q" value="<?= esc($q); ?>" class="form-control" placeholder="Cari judul artikel">
    </div>
    <div class="col-md-4">
        <select name="kategori_id" class="form-select">
            <option value="">Semua Kategori</option>
            <?php foreach ($kategori as $k): ?>
                <option value="<?= esc($k['id_kategori']); ?>" <?= ($kategori_id == $k['id_kategori']) ? 'selected' : ''; ?>>
                    <?= esc($k['nama_kategori']); ?>
                </option>
            <?php endforeach; ?>
        </select>
    </div>
    <div class="col-md-4">
        <button type="submit" class="btn btn-primary w-100">Cari</button>
    </div>
</form>

<!-- Tabel Artikel -->
<div class="table-responsive">
    <table class="table table-bordered table-hover align-middle">
        <thead class="table-light text-center">
            <tr>
                <th>ID</th>
                <th>Gambar</th>
                <th>Judul</th>
                <th>Kategori</th>
                <th>Status</th>
                <th>Aksi</th>
            </tr>
        </thead>
        <tbody>
            <?php if (!empty($artikel)): ?>
                <?php foreach ($artikel as $item): ?>
                    <tr>
                        <td class="text-center"><?= esc($item['id']); ?></td>
                        <td class="text-center">
                            <?php if (!empty($item['gambar'])): ?>
                                <img src="<?= base_url('gambar/' . $item['gambar']); ?>" 
                                     alt="<?= esc($item['judul']); ?>" 
                                     class="img-thumbnail" 
                                     style="width: 80px; height: 60px; object-fit: cover;">
                            <?php else: ?>
                                <span class="text-muted">No Image</span>
                            <?php endif; ?>
                        </td>
                        <td>
                            <strong><?= esc($item['judul']); ?></strong>
                            <p><small><?= esc(substr($item['isi'], 0, 50)); ?>...</small></p>
                        </td>
                        <td class="text-center"><?= esc($item['nama_kategori']); ?></td>
                        <td class="text-center">
                            <span class="badge bg-<?= $item['status'] ? 'success' : 'secondary'; ?>">
                                <?= $item['status'] ? 'Aktif' : 'Draft'; ?>
                            </span>
                        </td>
                        <td class="text-center">
                            <a href="<?= base_url('admin/artikel/edit/' . $item['id']); ?>" class="btn btn-sm btn-warning">Ubah</a>
                            <a href="<?= base_url('admin/artikel/delete/' . $item['id']); ?>" 
                               class="btn btn-sm btn-danger" 
                               onclick="return confirm('Yakin ingin menghapus artikel ini?');">
                               Hapus
                            </a>
                        </td>
                    </tr>
                <?php endforeach; ?>
            <?php else: ?>
                <tr>
                    <td colspan="6" class="text-center">Data tidak ditemukan.</td>
                </tr>
            <?php endif; ?>
        </tbody>
    </table>
</div>

<!-- Pagination -->
<div class="d-flex justify-content-start">
    <?= $pager->only(['q', 'kategori_id'])->links(); ?>
</div>

<?= $this->include('template/admin_footer'); ?>
```

## form_add.php
![{1A08CA5F-FDF5-416B-AC43-041B6C11FB5B}](https://github.com/user-attachments/assets/53077913-1531-4e56-b4f6-e1f11cff0925)


## form_edit.php
![{D7D4F2FA-98B3-403B-AC0F-901B722A7515}](https://github.com/user-attachments/assets/75a1f209-a5b0-402f-b862-abe6ca2faeb7)

##  Testing Fitur Relasi Artikel dan Kategori

Setelah seluruh konfigurasi selesai (tabel, model, controller, dan view), saatnya melakukan **pengujian** untuk memastikan bahwa relasi artikel ↔ kategori berjalan sebagaimana mestinya.
![{F4A8E7B4-3AF9-4DDE-ABD5-298EF893FF87}](https://github.com/user-attachments/assets/a8a216c6-da12-4305-aa8f-fc7f6ef884d5)

##  Testing: Menambah Artikel Baru dengan Memilih Kategori

Fitur ini menguji apakah proses input artikel baru berhasil menyimpan data **termasuk kategori yang dipilih**, serta apakah relasi artikel ↔ kategori bekerja sesuai harapan.
![{C063FC6A-5C9F-4C61-8EA6-C63C131E0A16}](https://github.com/user-attachments/assets/658dbdea-c6a9-4973-adaa-c92e2a69767c)

##  Testing: Mengedit Artikel dan Mengubah Kategorinya

Fitur ini menguji apakah admin dapat mengubah data artikel **termasuk mengganti kategori** yang telah dipilih sebelumnya.
![{10BF728C-A7CE-4033-95A3-733FDDFFC1C1}](https://github.com/user-attachments/assets/d78222ff-88bd-4c2b-84e3-8363359a8afa)

# Testing: Menghapus Artikel

Fitur ini menguji apakah admin dapat **menghapus artikel** yang sudah ada, dan memastikan data benar-benar terhapus dari database.
## Sebelum
![{97A04EB8-E2D1-41CE-BBF1-039DDEA4A98D}](https://github.com/user-attachments/assets/21c35dc0-109c-4d95-a266-83f58dff6b2b)

## Sesudah
![{387B9D09-87C8-45B8-A267-C0F6D947C6F3}](https://github.com/user-attachments/assets/4bf9d6c3-4eda-4c90-be77-a553ff859529)


##  Membuat AJAX Controller

AJAX Controller digunakan untuk menangani permintaan data secara dinamis dari **frontend** (seperti Vue.js atau jQuery), **tanpa perlu me-refresh halaman**. Umumnya digunakan untuk proses **CRUD** melalui RESTful API.
yaitu Ajax.php :
![{5A83BB07-9D44-41AB-8097-5736DCE6FBDA}](https://github.com/user-attachments/assets/cc11c7d1-3463-4b2b-bbc3-2a569e7e9fe7)

## Membuat View

Dalam arsitektur MVC CodeIgniter 4, **View** bertanggung jawab untuk menampilkan data yang dikirim oleh Controller ke pengguna. View dapat berupa HTML, dengan sedikit PHP untuk menampilkan variabel atau melakukan loop.
![{12340572-DC3A-4CEC-B5B6-702737997F7C}](https://github.com/user-attachments/assets/086172a5-1293-4bf3-9031-a28358c3a600)
![{37833F68-4D00-4E7D-B06B-21D6DA3CFB4A}](https://github.com/user-attachments/assets/36a41778-b0ff-4e4d-a35f-c76997421198)

## Persiapan

- Pastikan MySQL Server sudah berjalan.  
- Buka database `lab_ci4`.  
- Pastikan tabel `artikel` dan `kategori` sudah ada dan terisi data.  
- Pastikan library jQuery sudah terpasang atau dapat diakses melalui Content Delivery Network.

##  Modifikasi Controller Artikel

Ubah method `admin_index()` di `Artikel.php` untuk mengembalikan data dalam format JSON jika request adalah AJAX. (Sama seperti modul sebelumnya)

setelah saya upgrade kodenya :

```php
public function admin_index()
    {
        $title = 'Daftar Artikel';
        $model = new ArtikelModel();
        $kategoriModel = new KategoriModel();

        // Ambil keyword pencarian & filter kategori
        $q = $this->request->getGet('q') ?? '';
        $kategori_id = $this->request->getGet('kategori_id') ?? '';

        // Bangun query
        $builder = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left');

        if (!empty($q)) {
            $builder->like('artikel.judul', $q);
        }

        if (!empty($kategori_id)) {
            $builder->where('artikel.id_kategori', $kategori_id);
        }

        $data = [
            'title' => $title,
            'artikel' => $builder->paginate(10),
            'pager' => $model->pager,
            'q' => $q,
            'kategori_id' => $kategori_id,
            'kategori' => $kategoriModel->findAll(),
        ];

        return view('artikel/admin_index', $data);
    }
```
## 🛠️ Modifikasi Controller Artikel

Penjelasan:
• `$page = $this->request->getVar('page') ?? 1;`: Mendapatkan nomor halaman dari request. Jika tidak ada, default ke halaman 1.  
• `$builder->paginate(10, 'default', $page);`: Menerapkan pagination dengan nomor halaman yang diberikan.  
• `$this->request->isAJAX()`: Memeriksa apakah request yang datang adalah AJAX.  
• Jika AJAX, kembalikan data artikel dan pager dalam format JSON.  
• Jika bukan AJAX, tampilkan view seperti biasa.


# Pertanyaan dan Tugas

Selesaikan semua langkah praktikum di atas.  
Modifikasi tampilan data artikel dan pagination sesuai kebutuhan desain.
![{1F7E7A54-D651-4DEC-8178-517F14BB6392}](https://github.com/user-attachments/assets/f8584e80-40ba-45b6-98fd-473d476bba6c)

Implementasikan fitur sorting (mengurutkan artikel berdasarkan judul, dll.) dengan AJAX.  
- Berdasarkan Judul dan kategori :
![{E2781FE6-02C6-4753-A61D-AC3B7AE66F3C}](https://github.com/user-attachments/assets/40a15961-fdf1-4215-b544-b015fd1e8539)

- Berdasarkan Judul:
![{5666588C-9A8D-4CCD-A782-DA09C26911F1}](https://github.com/user-attachments/assets/469ebb77-5cd2-4aa1-bc06-ae9b2b749b9d)

- Berdasarkan Kategori
![{98802889-33EB-4A4C-9C42-F056B0B687F5}](https://github.com/user-attachments/assets/aabee613-7f82-4b75-a221-5a0022aef72c)
