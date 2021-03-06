<template>
    <div class="agile" :class="{'agile--fade': fade_}">
        <div class="agile__list">
            <div class="agile__track"
                 :style="{width: width.track + 'px', transform: 'translate(-' + transform + 'px)', transition: 'transform ' + timing_ + ' ' + transitionDelay + 'ms'}">
                <slot></slot>
            </div>

            <ul v-if="dots_" class="agile__dots">
                <li v-for="n in slidesCount" class="agile__dot"
                    :class="{'agile__dot--current': n - 1 === currentSlide}">
                    <button @click="setSlide(n - 1)">{{n}}</button>
                </li>
            </ul>

            <button v-if="arrows_" class="agile__arrow agile__arrow--prev"
                    :disabled="currentSlide === 0 && !infinite_" @click="prevSlide" v-html="arrow_">
            </button>
            <button v-if="arrows_" class="agile__arrow agile__arrow--next"
                    :disabled="currentSlide === slidesCount - 1 && !infinite_" @click="nextSlide" v-html="arrow_">
            </button>
        </div>
    </div>
</template>

<script>
    export default {
        name: 'agile',

        props: {
            arrow: {
                type: String,
                default: '<svg version="1.1" id="arrow-svg" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" viewBox="0 0 240.823 240.823" style="enable-background:new 0 0 240.823 240.823;" xml:space="preserve"><g><path id="arrow" d="M183.189,111.816L74.892,3.555c-4.752-4.74-12.451-4.74-17.215,0c-4.752,4.74-4.752,12.439,0,17.179 l99.707,99.671l-99.695,99.671c-4.752,4.74-4.752,12.439,0,17.191c4.752,4.74,12.463,4.74,17.215,0l108.297-108.261 C187.881,124.315,187.881,116.495,183.189,111.816z"/></g></svg>'
            },

            arrows: {
                type: Boolean,
                default: true
            },

            autoplay: {
                type: Boolean,
                default: false
            },

            autoplaySpeed: {
                type: Number,
                default: 3000
            },

            dots: {
                type: Boolean,
                default: true
            },

            fade: {
                type: Boolean,
                default: false
            },

            infinite: {
                type: Boolean,
                default: true
            },

            pauseOnDotsHover: {
                type: Boolean,
                default: false
            },

            pauseOnHover: {
                type: Boolean,
                default: true
            },

            speed: {
                type: Number,
                default: 300
            },

            timing: {
                type: String,
                default: 'ease' // linear, ease-in, ease-out, ease-in-out
            }
        },

        data () {
            return {
                el: {
                    dots: null,
                    list: null,
                    track: null,
                    slides: null
                },
                slidesCount: 0,
                autoplayTimeout: null,
                allSlidesCount: 0,
                currentSlide: 0,
                mouseDown: false,
                dragStartX: 0,
                dragStaryY: 0,
                dragDistance: 0,
                swipeDistance: 50,
                transform: 0,
                transitionDelay: 0,
                width: {
                    document: 0,
                    container: 0,
                    slide: 0,
                    track: 0
                },
                slidesToShow: 1,
                arrow_: this.arrow,
                arrows_: this.arrows,
                autoplay_: this.autoplay,
                autoplaySpeed_: this.autoplaySpeed,
                dots_: this.dots,
                fade_: this.fade,
                infinite_: this.infinite,
                pauseOnDotsHover_: this.pauseOnDotsHover,
                pauseOnHover_: this.pauseOnHover,
                speed_: this.speed,
                timing_: this.timing
            }
        },

        mounted () {
            // Protection against contradictory settings
            if (this.autoplay_) {
                this.infinite_ = true
            }

            if (this.pauseOnDotsHover_) {
                this.dots_ = true
            }

            // Prepare list
            this.el.list = this.$el.getElementsByClassName('agile__list')[0]

            // Prepare dots
            if (this.dots_) {
                this.el.dots = this.$el.getElementsByClassName('agile__dots')[0].children
            }

            // Prepare slides
            this.el.slides = this.$el.getElementsByClassName('agile__track')[0].children
            this.slidesCount = this.el.slides.length

            if (this.infinite_ && !this.fade_) {
                this.allSlidesCount = this.slidesCount + 2
            } else {
                this.allSlidesCount = this.slidesCount
            }

            for (let i = 0; i < this.slidesCount; ++i) {
                this.el.slides[i].classList.add('agile__slide')

                // Prepare slides for fade mode
                if (this.fade_) {
                    this.el.slides[i].style.transition = 'opacity ' + this.timing + ' ' + this.speed + 'ms'
                }
            }

            // Prepare track
            this.el.track = this.$el.getElementsByClassName('agile__track')[0]

            // Prepare infinity mode
            if (this.infinite_ && !this.fade_) {
                let firstSlide = this.el.track.firstChild.cloneNode(true)
                let lastSlide = this.el.track.lastChild.cloneNode(true)

                firstSlide.classList.add('agile__slide--cloned')
                lastSlide.classList.add('agile__slide--cloned')

                this.el.track.insertBefore(lastSlide, this.el.slides[0])
                this.el.track.insertBefore(firstSlide, this.el.slides[this.slidesCount].nextSibling)
            }

            // Prepare autoplay
            if (this.autoplay_) {
                this.startAutoplay()
            }

            // Windows resize listener
            window.addEventListener('resize', this.getWidth)

            // Get width on start
            this.getWidth()

            // Mouse and touch events
            if ('ontouchstart' in window) {
                this.el.track.addEventListener('touchstart', this.handleMouseDown)
                this.el.track.addEventListener('touchend', this.handleMouseUp)
                this.el.track.addEventListener('touchmove', this.handleMouseMove)
            } else {
                this.el.track.addEventListener('mousedown', this.handleMouseDown)
                this.el.track.addEventListener('mouseup', this.handleMouseUp)
                this.el.track.addEventListener('mousemove', this.handleMouseMove)
            }

            // Autoplay
            if (this.autoplay_) {
                if (this.pauseOnHover_) {
                    this.el.track.addEventListener('mouseover', this.stopAutoplay)
                    this.el.track.addEventListener('mouseout', this.startAutoplay)
                }

                if (this.pauseOnDotsHover_) {
                    for (let i = 0; i < this.slidesCount; ++i) {
                        this.el.dots[i].addEventListener('mouseover', this.stopAutoplay)
                        this.el.dots[i].addEventListener('mouseout', this.startAutoplay)
                    }
                }
            }
        },

        beforeDestroy () {
            window.removeEventListener('resize', this.getWidth)

            if ('ontouchstart' in window) {
                this.el.track.removeEventListener('touchstart', this.handleMouseDown)
                this.el.track.removeEventListener('touchend', this.handleMouseUp)
                this.el.track.removeEventListener('touchmove', this.handleMouseMove)
            } else {
                this.el.track.removeEventListener('mousedown', this.handleMouseDown)
                this.el.track.removeEventListener('mouseup', this.handleMouseUp)
                this.el.track.removeEventListener('mousemove', this.handleMouseMove)
            }

            if (this.autoplay_) {
                if (this.pauseOnHover_) {
                    this.el.track.removeEventListener('mouseover', this.stopAutoplay)
                    this.el.track.removeEventListener('mouseout', this.startAutoplay)
                }

                if (this.pauseOnDotsHover_) {
                    for (let i = 0; i < this.slidesCount; ++i) {
                        this.el.dots[i].removeEventListener('mouseover', this.stopAutoplay)
                        this.el.dots[i].removeEventListener('mouseout', this.startAutoplay)
                    }
                }
            }
        },

        methods: {
            getWidth () {
                this.width = {
                    document: window.innerWidth || document.documentElement.clientWidth || document.body.clientWidth,
                    container: this.el.list.clientWidth,
                    slide: this.el.list.clientWidth / this.slidesToShow
                }
            },

            handleMouseDown (e) {
                if (!e.touches) {
                    e.preventDefault()
                }

                this.mouseDown = true
                this.dragStartX = ('ontouchstart' in window) ? e.touches[0].clientX : e.clientX
                this.dragStartY = ('ontouchstart' in window) ? e.touches[0].clientY : e.clientY
            },

            handleMouseMove (e) {
                let positionX = ('ontouchstart' in window) ? e.touches[0].clientX : e.clientX
                let positionY = ('ontouchstart' in window) ? e.touches[0].clientY : e.clientY

                let dragDistanceX = Math.abs(positionX - this.dragStartX)
                let dragDistanceY = Math.abs(positionY - this.dragStartY)

                if (dragDistanceX > 3 * dragDistanceY) {
                    this.dragDistance = positionX - this.dragStartX
                    this.disableScroll()
                }
            },

            handleMouseUp () {
                this.mouseDown = false
                this.enableScroll()
            },

            disableScroll () {
                document.ontouchmove = function (e) {
                    e.preventDefault()
                }
            },

            enableScroll () {
                document.ontouchmove = function (e) {
                    return true
                }
            },

            addActiveClass (i) {
                this.el.slides[i].classList.add('agile__slide--active')
            },

            startAutoplay () {
                this.autoplayTimeout = setTimeout(() => {
                    this.nextSlide()
                }, this.autoplaySpeed)
            },

            stopAutoplay () {
                clearTimeout(this.autoplayTimeout)
            },

            setSlide (n, transition = true, autoplayTimeout = true) {
                // Reset autoplay timeout and set new
                if (this.autoplay_ && autoplayTimeout) {
                    this.stopAutoplay()
                    this.startAutoplay()
                }

                if (this.fade_) {
                    // Disable transition for initial slide
                    if (transition === false) {
                        this.el.slides[n].style.transition = '0ms'

                        setTimeout(() => {
                            this.el.slides[n].style.transition = 'opacity ' + this.timing + ' ' + this.speed + 'ms'
                        }, 10)
                    }

                    for (let i = 0; i < this.allSlidesCount; ++i) {
                        this.el.slides[i].classList.remove('agile__slide--expiring')
                    }

                    if (this.infinite_ && n < 0) {
                        n = this.slidesCount - 1
                    } else if (n >= this.slidesCount) {
                        n = 0
                    }

                    // Set current slide as expiring
                    let e = this.currentSlide
                    this.el.slides[e].classList.add('agile__slide--expiring')

                    setTimeout(() => {
                        this.el.slides[e].classList.remove('agile__slide--expiring')
                    }, this.speed_)

                    this.transform = 0
                } else {
                    this.transform = n * this.width.slide
                }

                for (let i = 0; i < this.allSlidesCount; ++i) {
                    this.el.slides[i].classList.remove('agile__slide--active')
                }

                if (this.infinite_ && !this.fade_) {
                    this.transform += this.width.slide
                    this.addActiveClass(n + 1)
                } else {
                    this.addActiveClass(n)
                }

                if (!transition) {
                    this.transitionDelay = 0
                } else {
                    this.transitionDelay = this.speed
                }

                if (this.infinite_ && n < 0) {
                    this.currentSlide = this.slidesCount - 1

                    setTimeout(() => {
                        this.setSlide(this.slidesCount - 1, false)
                    }, this.speed)
                } else if (this.infinite_ && n >= this.slidesCount) {
                    this.currentSlide = 0

                    setTimeout(() => {
                        this.setSlide(0, false)
                    }, this.speed_)
                } else {
                    this.currentSlide = n
                }
            },

            nextSlide () {
                this.setSlide(this.currentSlide + 1)
            },

            prevSlide () {
                this.setSlide(this.currentSlide - 1)
            }
        },

        watch: {
            width () {
                // Actions on document resize
                for (let i = 0; i < this.allSlidesCount; ++i) {
                    this.el.slides[i].style.width = this.width.container + 'px'

                    // Prepare slides for fade mode
                    if (this.fade_) {
                        this.el.slides[i].style.transform = 'translate(-' + i * this.width.slide + 'px)'
                    }
                }

                // Prepare track
                this.width.track = this.width.container * this.allSlidesCount
                this.setSlide(this.currentSlide, false, false)
            },

            dragDistance () {
                if (!this.mouseDown) {
                    return
                }

                if (this.dragDistance > this.swipeDistance) {
                    if (!this.infinite_ && this.currentSlide === 0) {
                        return
                    }

                    this.prevSlide()
                    this.handleMouseUp()
                }

                if (this.dragDistance < -1 * this.swipeDistance) {
                    if (!this.infinite_ && this.currentSlide === this.slidesCount - 1) {
                        return
                    }

                    this.nextSlide()
                    this.handleMouseUp()
                }
            }
        }
    }
</script>

<style lang="scss" type="text/scss">
    .agile {
        &, * {
            &:focus,
            &:active {
                outline: none;
            }
        }

        &__list {
            display: block;
            margin: 0;
            overflow: hidden;
            padding: 0;
            position: relative;
            width: 100%;
        }

        &__track {
            align-items: center;
            display: flex;
            justify-content: flex-start;
        }

        &__slide {
            display: block;

            .agile--fade & {
                opacity: 0;
                position: relative;
                z-index: 0;

                &--active {
                    opacity: 1;
                    z-index: 2;
                }

                &--expiring {
                    opacity: 1;
                    transition-duration: 0s;
                    z-index: 1;
                }
            }
        }

        &__arrow {
            border: none;
            bottom: 10px;
            margin: 0;
            padding: 0;
            position: absolute;
            transition-duration: .3s;

            &[disabled] {
                cursor: default;
                opacity: .4;
            }

            &:hover {
                #arrow-svg {
                    fill: #333;
                }
            }

            &--prev {
                left: 0;

                #arrow-svg {
                    transform: scale(-1, 1);
                }
            }

            &--next {
                right: 0;
            }

            #arrow-svg {
                fill: #888;
                height: 20px;
                transition-duration: .3s;
            }
        }

        &__dots {
            align-items: center;
            display: flex;
            justify-content: center;
            list-style: none;
            margin: 20px 0;
            padding: 0;
            text-align: center;
            white-space: nowrap;
        }

        &__dot {
            margin: 0 10px;

            button {
                background-color: #eee;
                border: none;
                border-radius: 50%;
                cursor: pointer;
                display: block;
                height: 10px;
                font-size: 0;
                line-height: 0;
                margin: 0;
                transition-duration: .3s;
                width: 10px;
            }

            &--current,
            &:hover {
                button {
                    background-color: #888;
                }
            }
        }
    }
</style>
