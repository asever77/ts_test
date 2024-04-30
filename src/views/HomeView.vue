<template>
  <div class="mdl-drag" data-line-id="test">
    <button type="button" class="mdl-drag-item n1" data-line-object="1">1</button>
    <button type="button" class="mdl-drag-item n2" data-line-object="2">2</button>
    <button type="button" class="mdl-drag-item n3" data-line-object="3">3</button>

    <button type="button" class="mdl-drag-item n4" data-line-target="3">3</button>
    <button type="button" class="mdl-drag-item n5" data-line-target="2">2</button>
    <button type="button" class="mdl-drag-item n6" data-line-target="1">1</button>
  </div>
</template>

<script lang="ts">
interface DragLineOption {
  id: string;
  answer: number;
  callback: (info: CallbackInfo) => void;
  callbackComplete?: (info: CallbackInfo) => void;
  callbackCheck?: (info: CallbackInfo) => void;
}

interface CallbackInfo {
  answer_all_sum: number;
  answer_current_sum: number;
  answer_current?: string;
  answer_state?: boolean;
  answer_all_state?: boolean;
}

class DragLine {
  private id: string;
  private doc: Document;
  private wrap: HTMLElement;
  /* eslint-disable */
  private items: NodeListOf<HTMLButtonElement>;
  private objects: NodeListOf<HTMLButtonElement>;
  /* eslint-disable */
  private type: string;
  private wrap_t: number;
  private wrap_l: number;
  private wrap_w: number;
  private wrap_h: number;
  private n: number;
  private svg: SVGSVGElement | null;
  private answer_len: number;
  private answer_n: number;
  private complete_n: number;
  private callback: (info: CallbackInfo) => void;
  private callbackComplete?: (info: CallbackInfo) => void;
  private callbackCheck?: (info: CallbackInfo) => void;

  constructor(opt: DragLineOption) {
    console.log(opt);
    this.id = opt.id;
    this.doc = document;
    this.wrap = document.querySelector<HTMLElement>(`[data-line-id="${this.id}"]`) as HTMLElement;
    this.items = this.wrap.querySelectorAll<HTMLButtonElement>(`[data-line-object], [data-line-target]`);
    this.objects = this.wrap.querySelectorAll<HTMLButtonElement>(`[data-line-object]`);
    this.type = this.wrap.dataset.lineType ? this.wrap.dataset.lineType : 'single';

    const wrapRect = this.wrap.getBoundingClientRect();
    this.wrap_t = wrapRect.top;
    this.wrap_l = wrapRect.left;
    this.wrap_w = this.wrap.offsetWidth;
    this.wrap_h = this.wrap.offsetHeight;

    this.n = this.objects.length;
    this.svg = null;
    this.answer_len = opt.answer;
    this.answer_n = 0;
    this.complete_n = 0;

    this.callback = opt.callback;
    this.callbackComplete = opt.callbackComplete;
    this.callbackCheck = opt.callbackCheck;
  }

  init(): void {
    this.wrap.insertAdjacentHTML('beforeend', '<svg></svg>');
    this.svg = this.wrap.querySelector<SVGSVGElement>('svg');

    const set = (): void => {
      this.reset();
      const wrapRect = this.wrap.getBoundingClientRect();
      this.wrap_t = wrapRect.top;
      this.wrap_l = wrapRect.left;
      this.wrap_w = this.wrap.offsetWidth;
      this.wrap_h = this.wrap.offsetHeight;

      for (let i = 0; i < this.items.length; i++) {
        const item = this.items[i];
        const rect_item = item.getBoundingClientRect();
        const item_w = item.offsetWidth / 2;
        const item_h = item.offsetHeight / 2;

        item.dataset.name = String(i);
        item.dataset.x = String(rect_item.left + item_w - this.wrap_l);
        item.dataset.y = String(rect_item.top + item_h - this.wrap_t);
      }

    }
    set();

    const resizeObserver = new ResizeObserver(() => {
      set();
    });
    resizeObserver.observe(this.wrap);

    const actStart = (e: MouseEvent | TouchEvent): void => {
      console.log('actStart', this.wrap.querySelector('svg'));
      this.wrap.querySelector('svg')?.insertAdjacentHTML('beforeend', `<line x1="0" x2="0" y1="0" y2="0" data-state="ing"></line>`);
      this.wrap_t = this.wrap.getBoundingClientRect().top;
      this.wrap_l = this.wrap.getBoundingClientRect().left;
      this.wrap_w = this.wrap.offsetWidth;
      this.wrap_h = this.wrap.offsetHeight;

      const el_line = this.svg?.querySelector('line[data-state="ing"]') as SVGSVGElement;
      const el_item = e.currentTarget as HTMLDivElement;

      const rect_item = el_item.getBoundingClientRect();
      const is_object = el_item.dataset.lineObject ? true : false;
      const data_name = el_item.dataset.name;
      const value = is_object ? el_item.dataset.lineObject : el_item.dataset.lineTarget;
      const item_w = el_item.offsetWidth / 2;
      const item_h = el_item.offsetHeight / 2;
      const x_value: string = (rect_item.left + item_w - this.wrap_l) + '';
      const y_value: string = (rect_item.top + item_h - this.wrap_t) + '';
      let _x: number;
      let _y: number;

      el_line.setAttribute('x1', x_value);
      el_line.setAttribute('y1', y_value);
      el_line.setAttribute('x2', x_value);
      el_line.setAttribute('y2', y_value);

      const actEnd = () => {
        console.log('actEnd');
        const v_x = _x - this.wrap_l;
        const v_y = _y - this.wrap_t;
        let is_complete = false;
        let is_answer = false;

        el_line.dataset.state = 'complete';

        for (let item of this.items) {
          // item.dataset.state = '';
          const _is_object = item.dataset.lineObject ? true : false;

          //완료된아이템여부 확인
          const _is_complete = item.dataset.complete;
          const _value = _is_object ? item.dataset.lineObject : item.dataset.lineTarget;
          const _rect_item = item.getBoundingClientRect();
          const i_x = Number(item.dataset.x);
          const i_y = Number(item.dataset.y);
          const if_x = (v_x <= i_x + item_w && v_x + (item_w * 2) >= i_x + item_w);
          const if_y = (v_y >= i_y - item_h && v_y <= i_y - item_h + (item_h * 2));
          let is_selected = false;
          let connect_array;
          //1:1규칙용
          //특정영역안에 있고, 같은 분류가 아니며, 완료되지않은 아이템
          const if_compete = this.type === 'single' ?
            (if_x && if_y && is_object !== _is_object && !_is_complete) :
            (if_x && if_y && is_object !== _is_object);

          //1:n인 경우 이미 연결된 아이템 제외
          if (this.type === 'multiple' && item.dataset.connect) {
            connect_array = item.dataset.connect.split(',');

            for (let i = 0; i < connect_array.length; i++) {
              if (data_name === connect_array[i]) {
                is_selected = true;
                break;
              }
            }
          }

          //연결성공
          if (if_compete && !is_selected) {
            //연결된 아이템 완료상태
            el_item.dataset.complete = 'true';
            item.dataset.complete = 'true';
            //완료갯수
            this.complete_n = this.complete_n + 1;
            //아이템 연결된 정보
            (!item.dataset.connect) ?
              item.dataset.connect = el_item.dataset.name :
              item.dataset.connect = item.dataset.connect + ',' + el_item.dataset.name;
            //최종 라인종료 위치
            el_line.setAttribute('x2', (_rect_item.left + item_w - this.wrap_l) + '');
            el_line.setAttribute('y2', (_rect_item.top + item_h - this.wrap_t) + '');
            //정오답적용
            const v1: string[] = value?.split(',')!;
            const v2: string[] = _value?.split(',')!;
            const is_OX = (v: boolean) => {
              if (v) {
                el_line.dataset.answer = 'true';
                is_answer = true;
                this.answer_n = this.answer_n + 1;
              } else {
                el_line.dataset.answer = 'false';
                is_answer = false;
                this.type === 'multiple' ? this.answer_n = this.answer_n - 1 : '';
              }
            }
            (this.type === 'multiple') ?
              //multiple인 경우 정오답
              is_OX(v1.filter(x => v2.includes(x)).length > 0) :
              //single인 경우 정오답
              is_OX(value === _value);

            is_complete = true;

            break;
          } else {
            console.log('실패');
          }
        }

        (!is_complete) && el_line.remove();

        this.doc.removeEventListener('mousemove', actMove);
        this.doc.removeEventListener('mouseup', actEnd);
        this.doc.removeEventListener('touchmove', actMove);
        this.doc.removeEventListener('touchend', actEnd);
        this.callback({
                    /*전체정답갯수*/answer_all_sum: this.answer_len,
                    /*현재정답갯수*/answer_current_sum: this.answer_n,
                    /*선택한정답  */answer_current: value,
                    /*정오답상태  */answer_state: is_answer
        });
        this.complete_n === this.n && this.completeCallback();
      }
      const actMove = (e: any) => {
        console.log('actMove');
        _x = e.clientX ? e.clientX : e.targetTouches[0].clientX;
        _y = e.clientY ? e.clientY : e.targetTouches[0].clientY;
        el_line.setAttribute('x2', (_x - this.wrap_l) + '');
        el_line.setAttribute('y2', (_y - this.wrap_t) + '');
      }

      this.doc.addEventListener('mousemove', actMove);
      this.doc.addEventListener('mouseup', actEnd);
      this.doc.addEventListener('touchmove', actMove);
      this.doc.addEventListener('touchend', actEnd);
    }

    this.items.forEach(item => {
      item.addEventListener('mousedown', actStart);
      item.addEventListener('touchstart', actStart, { passive: true });
    });
  }
  private completeCallback(): void {
    this.callbackComplete && this.callbackComplete({
      /*전체정답갯수  */answer_all_sum: this.answer_len,
      /*현재정답갯수  */answer_current_sum: this.answer_n,
      /*전체정오답상태*/answer_all_state: this.answer_len === this.answer_n ? true : false
    });
  }

  //초기화 실행
  reset(): void {
    for (let item of this.items) {
      item.removeAttribute('data-state');
      item.removeAttribute('data-complete');
      item.removeAttribute('data-connect');
    }

    if (this.svg?.lastChild) {
      while (this.svg.lastChild) {
        this.svg.removeChild(this.svg.lastChild);
      }
    }
    this.wrap.dataset.state = "";
    this.complete_n = 0;
    this.answer_n = 0;
  }
  //정오답체크
  check(): void {
    this.wrap.dataset.state = "check";
    this.callbackCheck && this.callbackCheck({
            /*전체정답갯수  */answer_all_sum: this.answer_len,
            /*현재정답갯수  */answer_current_sum: this.answer_n,
            /*전체정오답상태*/answer_all_state: this.answer_len === this.answer_n ? true : false
    });
  }
  //정답확인
  complete(): void {
    this.reset();
    for (let i = 0; i < this.n; i++) {
      const el_object = this.items[i];
      const value = el_object.dataset.lineObject!;
      if (value !== 'null') {
        const _v = value.split(',');
        console.log(_v);
        for (let j = 0; j < _v.length; j++) {
          const el_target = this.wrap.querySelector<HTMLElement>(`[data-line-target="${_v[j]}"]`);

          this.svg?.insertAdjacentHTML('beforeend', `<line x1="${el_object.dataset.x}" x2="${el_target?.dataset.x}" y1="${el_object.dataset.y}" y2="${el_target?.dataset.y}" data-state="complete"></line>`);
        }
      }
    }
    this.answer_n = this.answer_len;
    this.wrap.dataset.state = "complete";
    this.completeCallback();
  }
}


setTimeout(() => {
  const dragline = new DragLine({
    id: 'test',
    answer: 3,
    callback: (v) => {
      //개별 이벤트 완료 시
      console.log('callback', v);
    },
    callbackComplete: (v) => {
      //전체 이벤트 완료 시
      console.log('callbackComplete', v);
    },
    callbackCheck: (v) => {
      //체크 이벤트 실행 시
      console.log('callbackCheck', v);
    }
  });
  dragline.init();
},1000 )

</script>


<style lang="scss">

[data-line-id] {
  position: relative;

  svg {
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: none;
  }

  line {
    pointer-events: none;
  }

  &[data-state="check"] {
    line[data-answer="false"] {
      stroke: red;
    }

    line[data-answer="true"] {
      stroke: blue;
    }
  }

  &[data-line-type="multiple"] {

    [data-line-object],
    [data-line-target] {
      &[data-complete="true"] {
        pointer-events: inherit;
      }

      &[data-state="none"] {
        pointer-events: none;
      }
    }
  }

  [data-line-object],
  [data-line-target] {
    &[data-complete="true"] {
      pointer-events: none;
    }

    &[data-state="none"] {
      pointer-events: none;
    }
  }

}

[data-drag-id] {
  [data-drag-target] {
    &[data-drag-state="dragover"] {
      outline: 1px solid #000;
    }
  }

  [data-drag-object] {
    &[data-drag-state="dragging"] {
      opacity: 0.1;
    }
  }
}



.mdl-drag {
  margin: 0 0;
  width: 100%;
  height: 26rem;
  margin: 0 auto;
  outline: .1rem dashed silver;

  line {
    stroke: black;
    stroke-width: 4;
    stroke-linecap: round;
  }

  &-item {
    position: absolute;
    width: 4rem;
    height: 4rem;
    outline: .2rem solid black;
    border-radius: 50%;
    text-align: center;
    font-size: 1.4rem;
    display: flex;
    justify-content: center;
    align-items: center;
  }


  &-area {
    position: absolute;
    width: 10rem;
    height: 10rem;

    // &[data-drag-align="center"] {
    //     display: flex !important;
    //     justify-content: center !important;
    //     align-items: center !important;
    //     gap:1rem;

    //     .mdl-drag-drop {position: static; transform: translate(0, 0) !important;}
    // }
  }

  &-drop {
    position: absolute;
    width: 2rem;
    height: 2rem;
    background: #222;
    color: #fff;
    z-index: 1;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;

    &.active {
      box-shadow: 0 0.4rem 0.8rem rgba(0, 0, 0, 0.3);
      z-index: 10;
    }
  }

  &-answer {
    display: none;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    justify-content: center;
    align-items: center;
    gap: 1rem;
    pointer-events: none;

    .mdl-drag-drop {
      position: relative;
      transform: translate(0, 0) !important;
    }
  }

  &[data-state="complete"] .mdl-drag-answer {
    display: flex;
  }
}



.n1 {
  top: 1rem;
  left: 1rem
}

.n2 {
  top: 10rem;
  left: 1rem
}

.n3 {
  top: 20rem;
  left: 1rem
}

.n4 {
  top: 1rem;
  right: 1rem
}

.n5 {
  top: 10rem;
  right: 1rem
}

.n6 {
  top: 20rem;
  right: 1rem;
}
</style>
