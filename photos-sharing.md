# Photo sharing apps System Design

membuat design system untuk aplikasi sharing photo yang menampilkan list photo yang dipost oleh user. dan juga user bisa membuat postingan baru berupa poto dan captionnya.

![mockig](https://github.com/user-attachments/assets/db5d5835-539a-4eed-b73a-b7f12afdf39f)

## Requirement Exploration

**core features yang akan disupport ?**

- feed yang mengandung photo yang diposting oleh akun yang difollow user dan photo yang diupload user.
- upload, add caption dan apply filter sebelum diposting

**Pagination seperti apa yang cocok untuk feed nya ?**

- infinite scrolling

**ketika posting multiple foto di 1 post, apakah captionnya untuk masing2 foto atau semua foto 1 caption ?**

- hanya 1 caption untuk semua photo

**device apa yang digunakan untuk aplikasi ini ?**

- fokus utama mobile, tapi harusnya bisa digunkana di desktop maupun tablet juga

## Arch / High level design

### Rendering approach

- interaksi di aplikasi lebih banyak di fitur pembuatan postingan dan fungsi like/coment pada postingan
- initial loading cepat dibutuhkan

### Arch Diagram

![Tanpa judul-2024-08-27-1755](https://github.com/user-attachments/assets/c4e9b68f-64fe-4ee9-99b8-332ef7d23e75)

- **Server**, provide HTTP API fetch feed posts, upload images, dan create posts
- **Controller**, control data flow (bisnis flow) di aplikasi dan melakukan network request ke server
- **Client Sore,** menyimpan data yang dibutuhkan dengan scope global di aplikasi. (session login user, data user)
- **Feed UI**, list photo yang dipost dan UI untuk create postingan
  - **image post**, list photo yang dipost
  - **post composer**, UI untuk upload photo, apply filter di foto, dan menambahkan caption untuk photo yang akan dipost ke server.

## Data Model

mostly data didapatkan dari server, seperti feed photo. yang dari client hanya photo yang diupload dan caption postingan.

| Entity    | Source              | Belongs to       | Fields                                                                |
| --------- | ------------------- | ---------------- | --------------------------------------------------------------------- |
| `Feed`    | Server              | Feed UI          | `posts` (list `Post`), `pagination`                                   |
| `Post`    | Server              | Feed Post        | `id`, `created_time`, `caption`, `image`, `author` (`User`), `images` |
| `User`    | Server              | Client Store     | `id`, `name`, `profile_photo_url`                                     |
| `NewPost` | User Input (Client) | Post Composer UI | `caption`, `images` (`Image`)                                         |
| `Image`   | Server/client       | Multiple         | `url`, `alt`, `width`, `height`                                       |

## API

### Feed list API

pagination menggunakan cursor based.

### Post Creation API

- user select photo di device mereka
- user bisa melakukan edit sederhana photo mereka. (resize, crop, filter)
- user menambahkan caption di postingannya
- user submit postingannya

menggunakan api terpisah antara upload image dan create post. agar bisa melakukan proses upload image di background, ketika membuat caption.

![flow upload](https://github.com/user-attachments/assets/9a141d2a-acde-4eb9-9471-501c9a25fc3a)

## Optimisasi

### General Optimisasi

- code splitting javascript
  - lazy load page create post, karena initial load pasti di page feed.

### Feed Optimisasi dan deepdive

- infinite scrolling
  - kita bisa optimize loading time ketika fetch feed baru, perlu melakukan fetch sebelum user mencapai akhir dari list postingan. jadi ketika user mencapai akhir dari feed, user tidak melihat ada loading indikator ketika fetch. untuk implementasi infinite scrolling sendiri bisa pakai intersection observer API.
- virtualized list
  - mengurangi DOM yang diappend yang akan secara otomatis mengurangi memory usage.
  - menampilkan feed yang masuk di viewport. feed yang tidak masuk akan dihide.

### Image Carousel Optimisasi

- fetch 3 item awal. ketika user berada pada 2 item terakhir di carousel, baru fetch 3 image selanjutnya. dan begitu seterusnya.

### Rendering Images

- pakai image webp, provide lossless image compression
- harus ada `alt` pada tag `img`
  - improve accessibility dan SEO
  - bisa dibuat auto generate text untuk alt dengan memanfaat kan machine learning.
- image loading berdasarkan ukuran screen dari device.
  - mengirimkan dimensi browser ketika init request. setelah itu dari server akan menentukan image dengan ukuran berapa yang akan dikirimkan.
  - menggunakan atribut srcset pada tag img. mencantumkan kandidat gambar di masing2 resolusi atau lebar gambar.

```jsx
<img
  src="gambar-kecil.jpg"
  srcset="gambar-kecil.jpg 300w, gambar-sedang.jpg 600w, gambar-besar.jpg 1200w"
  sizes="(max-width: 600px) 300px, (max-width: 1200px) 600px, 1200px"
  alt="Contoh gambar"
>

```

- adaptive image loading berdasarkan network speed.
  - Good network
    - prefetch image yang tidak muncul di layar tapi akan masuk dalam area layar.
  - Bad network
    - memuat placeholder gambar dengan berukuran rendah. dan memberikan interaksi ke user jika user ingin me load gambar dengan resolusi normal.

## Image Editing

### Cropping Resizing

cropping dan resizing bisa dilakukan menggunakan HTML5 `canvas`

```jsx
<canvas id="myCanvas" width="200" height="200"></canvas>
<img id="sourceImage" src="your-image.jpg" alt="Source Image" style="display: none;">
<script>
  const canvas = document.getElementById('myCanvas');
  const ctx = canvas.getContext('2d');
  const image = document.getElementById('sourceImage');

  image.onload = function () {
    // Tentukan area cropping dari gambar sumber
    const cropX = 50;
    const cropY = 50;
    const cropWidth = 400;
    const cropHeight = 400;

    // Resize gambar yang sudah dicrop agar sesuai dengan canvas
    ctx.drawImage(
      image,
      cropX, cropY, cropWidth, cropHeight, // Area cropping
      0, 0, canvas.width, canvas.height    // Gambar hasil cropping di-resize ke ukuran canvas
    );
  };
</script>

```

- pertama kali memotong gambar dari koordinat dan ukuran tertentu (`cropX`, `cropY`, `cropWidth`, `cropHeight`).
- Kemudian, gambar yang sudah dipotong di-resize agar sesuai dengan ukuran canvas (`canvas.width`, `canvas.height`).

Setelah gambar diolah dalam canvas, Anda bisa mengekspor hasilnya menggunakan `canvas.toBlob()` untuk nantinya diupload dengan formData.

### FIlters

css provide `filter` property yang mana bisa mengimplementasikan fungsionaliti seperti `blur` `contrast` `hue` `sepia`. dengan mengkombinasikan fungsionaliti filter tersebut, filter seperti yang ada di instagram bisa diachieve di browser.

```css
.filter-1977 {
  filter: sepia(0.5) hue-rotate(-30deg) saturate(1.4);
}

.filter-brannan {
  filter: sepia(0.4) contrast(1.25) brightness(1.1) saturate(0.9) hue-rotate(-2deg);
}

// source: https://github.com/picturepan2/instagram.css
```

```html
<img src="your-image.jpg" class="filter-1977" alt="1977 Filter" />
<img src="your-image.jpg" class="filter-brannan" alt="Brannan Filter" />
```

| **Original**                                                                                                                                    | **1977**                                                                                                                                        | **Brannan**                                                                                                                                     |
| ----------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| <img width="151" alt="Screenshot 2024-09-24 at 13 52 47" src="https://github.com/user-attachments/assets/c8094403-033c-40a9-aa13-bab93247c517"> | <img width="151" alt="Screenshot 2024-09-24 at 13 52 47" src="https://github.com/user-attachments/assets/ec4ee797-56ed-4ad8-80b7-35625ef7fd06"> | <img width="151" alt="Screenshot 2024-09-24 at 13 53 50" src="https://github.com/user-attachments/assets/c367b484-baaa-4999-bc27-4814f785c869"> |

hasil dari filter bisa diconvert menjadi image blob sebelum nantinya dikirim ke server dan proses conversi nya sendiri bisa menggunakan library https://github.com/niklasvh/html2canvas. penjelasan simplenya library tersebut melakukan screenshoot pada tag HTML yang ditarget.
