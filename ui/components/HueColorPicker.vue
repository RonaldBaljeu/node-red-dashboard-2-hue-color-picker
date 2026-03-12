<template>
    <!-- Component must be wrapped in a block so props such as className and style can be passed in from parent -->
    <div className="hue-color-picker-wrapper">

        <div
          ref="hue-color-picker-container"
          class="hue-color-picker-container"
          @mousedown="startDrag"
          @touchstart.prevent="startDrag"
        >
          <canvas ref="hue-color-picker-canvas" class="hue-color-picker-canvas"></canvas>

          <div
            class="hue-color-picker-cursor"
            :style="cursorStyle"
          ></div>
        </div>

        <h1>Example Integrated Widget: {{ id }}</h1>

        <h2>Accessing Properties</h2>
        <p>
            Your Vue component will be rendered and passed a few built-in properties.
        </p>

        <h3><code>this.id</code></h3>
        <p>Each widget has a unique ID, which can be used when sending messages to Node-RED via <code>this.$socket.emit()</code>.</p>
        <pre>{{ id }}</pre>

        <h3><code>this.props</code></h3>
        <p>
            The <code>props</code> object contains the properties defined in the widget's configuration in Node-RED.
            It is a snapshot of the properties at the time the widget was loaded
        </p>
        <pre>{{ props }}</pre>

        <h3><code>this.state</code></h3>
        <p>
            The <code>state</code> object contains any properties that have been overridden at runtime (after the node is deployed).
            Your node should support override of configuration options via <code>msg.ui_update.&lt;property&gt;</code>
        </p>
        <pre>{{ state }}</pre>

        <h2>Dynamic Properties</h2>
        <p>We have setup a dynamic property <code>example</code> which can be updated from Node-RED.</p>
        <p>
            We've wrapped this into a computed variable and used the built-in <code>getProperty()</code>
            function which automatically checks both <code>props</code> and <code>state</code>.
        </p>
        <pre>{{ example }}</pre>

        <h2>Data Retention &amp; VueX Stores</h2>
        <p>Dashboard 2.0 has a built-in VueX datastore. This can be used to store (and retrieve) the latest received messages.</p>
        <p>Note: the vuex store is cleared on refresh of a screen, at which point, data will be loaded from the Node-RED datastore, should it be present.</p>
        <p>Send a message to this node in order to see the value here:</p>
        <pre>{{ messages && messages[id] ? messages[id] : 'No Data' }}</pre>
        <p>Note that it persists, even after refresh. This is because, in our <code>onInput</code> event handler in our <code>hue-color-picker.js</code> file, we store the message in the Node-RED datastore.</p>

        <!-- Note: We can use any Vuetify Components by default -->
        <h2>Styling with Vuetify &amp; CSS</h2>
        <p>
            We can define our own CSS within the widget's repository,
            and expose classes like <code class="hue-color-picker-class">hue-color-picker-class</code>
        </p>
        <p>
            Vuetify also comes with a handful of utility classes to assist with styling:
            <ul>
                <li v-for="guide in vuetifyStyles" :key="guide.label"><a :href="guide.url" target="_blank">{{ guide.label }}</a></li>
            </ul>
        </p>

        <h2>External Dependencies</h2>
        <p>
            Any additional libraries you require can be installed as in any other package.
            When the <code>npm run build</code> command is run, Vite will package everything
            up into the single <code>.umd.js</code> file.
        </p>
        <p>
            Here, we include the NPM package <code>to-title-case</code>, and bind it's
            functionality to a VueJS Computed Variable, which automatically re-calculates.
        </p>
        <p>
            <v-text-field v-model="input.title" type="text" />
            VueJS Computed Variable Output: <i>{{ titleCase }}</i>
        </p>
    </div>
</template>

<script>
import toTitleCase from 'to-title-case'
import { mapState } from 'vuex'

export default {
    name: 'HueColorPicker',
    inject: ['$socket', '$dataTracker'],
    props: {
        /* do not remove entries from this - Dashboard's Layout Manager's will pass this data to your component */
        id: { type: String, required: true },
        props: { type: Object, default: () => ({}) },
        state: { type: Object, default: () => ({ enabled: false, visible: false }) }
    },
    setup (props) {
        console.info('HueColorPicker setup with:', props)
    },
    data () {
        return {
            ctx: null,
            radius: 0,
            modelValue: {
                x: 0,
                y: 0
            },

            input: {
                title: 'some text here will base turned into title case.'
            },
            vuetifyStyles: [
                { label: 'Responsive Displays', url: 'https://vuetifyjs.com/en/styles/display/#display' },
                { label: 'Flex', url: 'https://vuetifyjs.com/en/styles/flex/' },
                { label: 'Spacing', url: 'https://vuetifyjs.com/en/styles/spacing/#how-it-works' },
                { label: 'Text & Typography', url: 'https://vuetifyjs.com/en/styles/text-and-typography/#typography' }
            ]
        }
    },
    computed: {
        cursorStyle() {
            const normX = (this.modelValue.x / 0.8) * 2 - 1;
            const normY = 1 - (this.modelValue.y / 0.9) * 2;

            return {
                left: `${this.radius + normX * this.radius}px`,
                top: `${this.radius + normY * this.radius}px`,
            };
        },

        titleCase () {
            return toTitleCase(this.input.title)
        },
        ...mapState('data', ['messages']),
        // wrap our "example" property in a computed property to ensure it re-calculates when the property changes
        example () {
            // use the globally available "getProperty" function
            return this.getProperty('example')
        }
    },
    created () {
        // setup our event handlers, and informs Node-RED that this widget has loaded
        this.$dataTracker(this.id, this.onInput, this.onLoad, this.onDynamicProperties)
    },
    mounted() {
        setTimeout(() => {
            this.drawPicker();
        }, 200);
    },
    methods: {


        // --------------------
        // Color Math
        // --------------------

        xyToRgb(x, y) {
            if (y === 0) return [0, 0, 0];

            const Y = 1;
            const X = (x * Y) / y;
            const Z = ((1 - x - y) * Y) / y;

            let r = 3.2406 * X - 1.5372 * Y - 0.4986 * Z;
            let g = -0.9689 * X + 1.8758 * Y + 0.0415 * Z;
            let b = 0.0557 * X - 0.2040 * Y + 1.0570 * Z;

            r = Math.max(0, r);
            g = Math.max(0, g);
            b = Math.max(0, b);

            const max = Math.max(r, g, b);
            if (max > 1) {
                r /= max;
                g /= max;
                b /= max;
            }

            const gamma = (c) =>
                c <= 0.0031308
                    ? 12.92 * c
                    : 1.055 * Math.pow(c, 1 / 2.4) - 0.055;

            return [
                Math.round(gamma(r) * 255),
                Math.round(gamma(g) * 255),
                Math.round(gamma(b) * 255),
            ];
        },
        xyToCanvas(x, y) {
            const normX = (x / 0.8) * 2 - 1;
            const normY = 1 - (y / 0.9) * 2;

            return {
                px: this.radius + normX * this.radius,
                py: this.radius + normY * this.radius,
            };
        },

        // --------------------
        // Render CIE picker
        // --------------------
        drawPicker() {
            const size = this.$refs['hue-color-picker-container'].offsetWidth;
            const canvas = this.$refs['hue-color-picker-canvas'];

            canvas.width = size;
            canvas.height = size;

            this.radius = size / 2;
            this.ctx = canvas.getContext("2d");

            const image = this.ctx.createImageData(size, size);
            const data = image.data;

            for (let py = 0; py < size; py++) {
                for (let px = 0; px < size; px++) {
                    const normX = (px - this.radius) / this.radius;
                    const normY = (py - this.radius) / this.radius;

                    const cieX = 0.8 * (normX + 1) / 2;
                    const cieY = 0.9 * (1 - (normY + 1) / 2);

                    const [r, g, b] = this.xyToRgb(cieX, cieY);

                    const index = (py * size + px) * 4;
                    data[index] = r;
                    data[index + 1] = g;
                    data[index + 2] = b;
                    data[index + 3] = 255;
                }
            }

            this.ctx.putImageData(image, 0, 0);
            this.drawGamutTriangle();
            this.drawWhitePoint();
        },
        drawGamutTriangle() {
            const GAMUT_C = {
                red: { x: 0.6915, y: 0.3083 },
                green: { x: 0.17, y: 0.7 },
                blue: { x: 0.1532, y: 0.0475 },
            };

            const r = this.xyToCanvas(GAMUT_C.red.x, GAMUT_C.red.y);
            const g = this.xyToCanvas(GAMUT_C.green.x, GAMUT_C.green.y);
            const b = this.xyToCanvas(GAMUT_C.blue.x, GAMUT_C.blue.y);

            this.ctx.beginPath();
            this.ctx.moveTo(r.px, r.py);
            this.ctx.lineTo(g.px, g.py);
            this.ctx.lineTo(b.px, b.py);
            this.ctx.closePath();

            this.ctx.strokeStyle = "black";
            this.ctx.lineWidth = 2;
            this.ctx.stroke();
        },
        drawWhitePoint() {
            const WHITE_POINT = { x: 0.3127, y: 0.329 };

            const { px, py } = this.xyToCanvas(
                WHITE_POINT.x,
                WHITE_POINT.y
            );

            this.ctx.beginPath();
            this.ctx.arc(px, py, 6, 0, Math.PI * 2);

            this.ctx.fillStyle = "white";
            this.ctx.fill();

            this.ctx.lineWidth = 2;
            this.ctx.strokeStyle = "black";
            this.ctx.stroke();
        },

        // --------------------
        // Interaction
        // --------------------
        updateFromEvent(event) {
            const canvas = this.$refs['hue-color-picker-canvas'];
            const rect = canvas.getBoundingClientRect();

            const clientX = event.touches
                ? event.touches[0].clientX
                : event.clientX;
            const clientY = event.touches
                ? event.touches[0].clientY
                : event.clientY;

            let px = Math.max(
                0,
                Math.min(clientX - rect.left, rect.width - 1)
            );
            let py = Math.max(
                0,
                Math.min(clientY - rect.top, rect.height - 1)
            );

            const normX = (px - this.radius) / this.radius;
            const normY = (py - this.radius) / this.radius;

            const cieX = 0.8 * (normX + 1) / 2;
            const cieY = 0.9 * (1 - (normY + 1) / 2);

            this.modelValue.x = cieX;
            this.modelValue.y = cieY;
            this.$socket.emit('widget-action', this.id, {
                payload: {
                    x: cieX,
                    y: cieY
                }
            });
        },
        startDrag(event) {
            this.updateFromEvent(event);

            const move = (e) => this.updateFromEvent(e);
            const stop = () => {
                window.removeEventListener("mousemove", move);
                window.removeEventListener("mouseup", stop);
                window.removeEventListener("touchmove", move);
                window.removeEventListener("touchend", stop);
            };

            window.addEventListener("mousemove", move);
            window.addEventListener("mouseup", stop);
            window.addEventListener("touchmove", move);
            window.addEventListener("touchend", stop);
        },

        /*
            widget-action just sends a msg to Node-RED, it does not store the msg state server-side
            alternatively, you can use widget-change, which will also store the msg in the Node's datastore
        */
        send (msg) {
            this.$socket.emit('widget-action', this.id, msg)
        },
        /*
            (optional) Custom onInput function to handle incoming messages from Node-RED
        */
        onInput (msg) {
            this.modelValue.x = msg.payload.x;
            this.modelValue.y = msg.payload.y;

            // load the latest message from the Node-RED datastore when this widget is loaded
            // storing it in our vuex store so that we have it saved as we navigate around
            this.$store.commit('data/bind', {
                widgetId: this.id,
                msg
            });
        },
        /*
            (optional) Custom onDynamicProperties function to handle dynamic properties
            msg - the latest message from the Node-RED datastore
        */
        onDynamicProperties (msg) {
            // handle any dynamic properties that are sent from Node-RED
            const updates = msg.ui_update
            if (!updates) {
                return
            }
            if (typeof updates.example !== 'undefined') {
                // use the globally available "setDynamicProperties" function to store any updates to this property
                this.setDynamicProperties({ example: updates.example })
            }
        }
    }
}
</script>

<style scoped>
.hue-color-picker-container {
    position: relative;
    width: 320px;
    height: 320px;
    user-select: none;
}

.hue-color-picker-canvas {
    width: 100%;
    height: 100%;
    display: block;
}

.hue-color-picker-cursor {
    position: absolute;
    width: 18px;
    height: 18px;
    border: 3px solid white;
    border-radius: 50%;
    transform: translate(-50%, -50%);
    box-shadow: 0 0 6px rgba(0, 0, 0, 0.6);
    pointer-events: none;
}
</style>
