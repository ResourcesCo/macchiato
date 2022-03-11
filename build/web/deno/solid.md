##### `build.js`

```ts
const example = `
import { createSignal, onCleanup } from "solid-js";
import { render } from "solid-js/web";

const CountingComponent = () => {
	const [count, setCount] = createSignal(0);
	const interval = setInterval(
		() => setCount(c => c + 1),
		1000
	);
	onCleanup(() => clearInterval(interval));
	return <div>Count value is {count()}</div>;
};

render(() => <CountingComponent />, document.getElementById("app"));
`;

import Babel from "https://esm.sh/@babel/standalone";
import solid from "https://esm.sh/babel-preset-solid";
Babel.registerPreset("solid", solid());
console.log(Babel.transform(example, {presets: ["solid"]}).code);
```