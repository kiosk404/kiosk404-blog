<div class="button-group">
    <button class="filter-button" data-category="-">&nbsp;All&nbsp;</button>
    <button class="filter-button" data-category="日常">#日常</button>
    <button class="filter-button" data-category="风景">#风景</button>
    <button class="filter-button" data-category="狗哥">#狗哥</button>
    <button class="filter-button" data-category="赛">#赛</button>
</div>
<div class="gallery-photos page">
 

<style>
.gallery-photos{width:100%;}
.gallery-photo{width:24.9%;position: relative;visibility: hidden;overflow: hidden;}
.gallery-photo.visible{visibility: visible;animation: fadeIn 2s;}
.gallery-photo img{
    display: block;
    width:100%;
    border-radius:20px;
    padding:4px;
    animation: fadeIn 1s;
    cursor: pointer;
    transition: all .4s ease-in-out;
}
.gallery-photo span.photo-title,.gallery-photo span.photo-time{
    background: rgba(0, 0, 0, 0.3);
    padding:0px 8px;
    font-size:0.9rem;
    color: #fff;
    display:none;
    animation: fadeIn 1s;
}
.gallery-photo span.photo-title{
    position:absolute;
    bottom:4px;
    left:4px;
    border-radius:10px;
    background-color: #a3dfff;
    color: #000000;
    font-size:1.2rem
}
.gallery-photo span.photo-time{position:absolute;top:4px;left:4px;font-size:0.8rem;}
.gallery-photo:hover span.photo-title{display:block;}
.gallery-photo:hover img{transform: scale(1.1);}
.button-group {
                display: flex;
                flex-direction: row;
                gap: 18px;
                justify-content: center;
            }
            .filter-button {
                padding: 8px 8px;
                width: auto;
                background-color: var(--card-background);
                color: #57bd8f;
                border: none;
                border-radius: 5px;
                cursor: pointer;
                transition: background-color 0.3s ease;
            }
            .filter-button:hover {
                color: #5e88f7;
            }
            .selected-button {
                background-color: #CCE8CF;
                color: #000000;
                /* 选中项的颜色 */
            }
@media screen and (max-width: 1280px) {
	.gallery-photo{width:33.3%;}
}
@media screen and (max-width: 860px) {
	.gallery-photo{width:49.9%;}
}
@media (max-width: 683px){
	.photo-time{display: none;}
}
@keyframes fadeIn{
	0% {opacity: 0;}
   100% {opacity: 1;}
}
</style>
<script src="https://immmmm.com/waterfall.min.js"></script>
<script src="https://immmmm.com/imgStatus.min.js"></script>
<script>
document.addEventListener('DOMContentLoaded', () => {
    imgStatus.watch('.photo-img', function(imgs) {
      if(imgs.isDone()){
        waterfall('.gallery-photos');
        let pagePhoto = document.querySelectorAll('.gallery-photo');
        for(var i=0;i < pagePhoto.length;i++){pagePhoto[i].className += " visible"};
      }
    });
    window.addEventListener('resize', function () {
      waterfall('.gallery-photos');
    });
});
</script>
<script src="https://immmmm.com/view-image.js"></script>
<script src="https://immmmm.com/lately.min.js"></script>
<script>
  window.Lately && Lately.init({ target: '.photo-time'});
  window.ViewImage && ViewImage.init('.gallery-photo img')
</script>
<script>
    document.addEventListener('DOMContentLoaded', () => {
    const filterButtons = document.querySelectorAll('.filter-button');
    const galleryPhotos = document.querySelectorAll('.gallery-photo');
    filterButtons.forEach(button => {
        button.addEventListener('click', () => {
            const category = button.getAttribute('data-category');
            // 隐藏所有照片
            galleryPhotos.forEach(photo => {
                photo.style.visibility = 'hidden';
            });
            // 显示符合特定词语的照片
            galleryPhotos.forEach(photo => {
                const imageName = photo.querySelector('.photo-img').getAttribute('alt');
                if (imageName.includes(category)) {
                    photo.style.visibility = 'visible';
                }
            });
            // 移除所有按钮的选中状态
            filterButtons.forEach(btn => {
                btn.classList.remove('selected-button');
            });
            // 将当前点击的按钮标记为选中状态
            button.classList.add('selected-button');
        });
    });
});
</script>
