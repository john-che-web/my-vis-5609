<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
    import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
    import type { TMovie } from "../../types";
    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";

    import * as d3 from "d3";

    let movies: TMovie[] = $state([]);

    // Stacked data: each year has counts per genre
    type TStackRow = {
        year: string;
        Comedy: number;
        Romance: number;
        Drama: number;
    };

    let stackData: TStackRow[] = $state([]);
    const genres = ["Comedy", "Romance", "Drama"] as const;
    const genreColors: Record<string, number> = {
        Comedy: 0xf4a261,  // warm orange
        Romance: 0xe76f51,  // coral red
        Drama: 0x2a9d8f,   // teal
    };

    // === TODO-1: Load Data ===
    // Currently using hardcoded dummy data.
    // TODO: Load the summer_movies.csv, then aggregate
    //   the real movie data into stackData.
    //   - Group movies by year (movie.year.getFullYear())
    //   - Count how many movies belong to each genre (Comedy, Romance, Drama)
    //   - Build a TStackRow[] array sorted by year
    async function loadCsv() {
        const rows = await d3.csv("/summer_movies.csv");
        const parsedMovies: TMovie[] = rows
            .map((row) => ({
                tconst: row.tconst ?? "",
                title_type: row.title_type ?? "",
                primary_title: row.primary_title ?? "",
                original_title: row.original_title ?? "",
                simple_title: row.simple_title ?? "",
                year: /^\d{4}$/.test(row.year ?? "")
                    ? new Date(Number(row.year), 0, 1)
                    : null,
                runtime_minutes: row.runtime_minutes ? Number(row.runtime_minutes) : null,
                genres: row.genres
                    ? row.genres.split(",").map((genre) => genre.trim())
                    : [],
                average_rating: row.average_rating ? Number(row.average_rating) : null,
                num_votes: row.num_votes ? Number(row.num_votes) : null,
            }))
            .filter((movie) => movie.year !== null);

        movies = parsedMovies;
        const grouped = d3.rollup(
            parsedMovies,
            (yearMovies) => {
                const counts = { Comedy: 0, Romance: 0, Drama: 0 };
                yearMovies.forEach((movie) => {
                    genres.forEach((genre) => {
                        if (movie.genres.includes(genre)) counts[genre] += 1;
                    });
                });
                return counts;
            },
            (movie) => movie.year!.getFullYear(),
        );

        stackData = Array.from(grouped, ([year, counts]) => ({
            year: String(year),
            ...counts,
        })).sort((a, b) => Number(a.year) - Number(b.year));

        console.log("Stack data:", stackData);
    }
    // === END TODO-1 ===

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;

    const morphs: Array<THREE.Mesh> = [];
    let mixer: THREE.AnimationMixer;
    const clock = new THREE.Clock();

    const horseMorphs: Array<THREE.Mesh> = [];
    const expressiveObjects: Array<THREE.Object3D> = [];
    let horseMixer: THREE.AnimationMixer;
    const ORBIT_RX = 850;
    const ORBIT_RZ = 400;


    let animating = $state(false);

    onMount(async () => {
        await loadCsv();
        init(window.innerWidth, window.innerHeight);
    });

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number) {
        // Renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        // Camera
        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            5000,
        );
        camera.position.set(0, 100, 2800);

        // Scene
        scene = new THREE.Scene();
        new THREE.TextureLoader().load("3d/sky.jpg", (texture) => {
            texture.repeat.set(0.8, 1);
            scene.background = texture;
        });

        // Ambient Light
        const ambient = new THREE.AmbientLight(0xffffff);
        scene.add(ambient);

        // Directional Light
        const light = new THREE.DirectionalLight(0xffffff, 3);
        light.position.set(0, 1500, 1000);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2000,
            bottom: -2000,
            left: -2000,
            right: 2000,
            near: 1200,
            far: 2500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        // Ground
        addGround(scene, FLOOR, "3d/grasslight-big.jpg");

        // Title + Stacked Bars
        const fontLoader = new FontLoader();
        fontLoader.load("3d/helvetiker_bold.typeface.json", (font: Font) => {
            // Title
            const textGeo = new TextGeometry("summer movies", {
                font: font,
                size: 30,
                depth: 10,
            });
            textGeo.computeBoundingBox();
            const centerOffset =
                -0.5 *
                (textGeo!.boundingBox!.max.x - textGeo!.boundingBox!.min.x);
            const textMaterial = new THREE.MeshStandardMaterial({
                color: 0x449900,
            });
            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 500;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            createUnits(scene, font);
            createLegend(scene, font);
        });

        // === TODO-2: Add Expressive 3D Objects ===
        // Make the scene more expressive, engaging and fun by changing or adding
        // at least one 3D object! You do NOT need to replace all existing objects —
        // modifying or adding just one is sufficient.
        // Animation is required (e.g., rotation, bobbing, or using a model's built-in animation clips).
        //
        // Where to find free GLB models:
        //   - https://poly.pizza (search, download as .glb)
        //   - https://sketchfab.com (filter by "Downloadable" + "glTF")
        //   - https://kenney.nl/assets (game-style assets, some 3D)
        //   - https://www.fab.com/search?q=3d+object&asset_formats=glb&asset_formats=converted-files&is_free=1
        //
        // Tips:
        //   - Place your .glb files in the /static/3d/ folder
        //   - Adjust the `scale` value if your model is too big or small
        //   - If a model has no built-in animations, add a custom one
        //     (e.g., rotate or float the mesh in the animate() function)
        //   - If a model has no animations, guard the animation code:
        //     if (gltf.animations.length > 0) { ... }
        //   - Feel free to add extra static models as decoration (trees, buildings, etc.)!

        // Load bird models (linear motion)
        const birdModels = [
            {
                path: "3d/Flamingo.glb", // TODO: replace with your model
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 0.5,
            },
            {
                path: "3d/Flamingo.glb", // TODO: replace with your model
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: -100,
                scale: 0.5,
            },
            {
                path: "3d/Parrot.glb", // TODO: replace with your model
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 0.5,
            },
        ];
        mixer = loadModels(birdModels, scene, mixer, morphs);

        // Load horse models (orbit around data)
        horseMixer = new THREE.AnimationMixer(scene);
        const gltfLoader = new GLTFLoader();
        const horseConfigs = [
            { startAngle: 0, orbitSpeed: 0.4 },
            { startAngle: Math.PI, orbitSpeed: 0.4 },
        ];
        horseConfigs.forEach((config) => {
            gltfLoader.load("3d/Horse.glb", (gltf) => { // TODO: replace with your model
                const mesh = gltf.scene.children[0].clone() as THREE.Mesh;
                (mesh as any).material = (mesh as any).material.clone();
                mesh.scale.set(0.5, 0.5, 0.5);
                mesh.castShadow = true;

                horseMixer
                    .clipAction(gltf.animations[0], mesh)
                    .setDuration(1)
                    .startAt(-Math.random())
                    .play();

                mesh.userData.angle = config.startAngle;
                mesh.userData.orbitSpeed = config.orbitSpeed;
                mesh.position.set(
                    ORBIT_RX * Math.cos(config.startAngle),
                    FLOOR,
                    ORBIT_RZ * Math.sin(config.startAngle),
                );

                scene.add(mesh);
                horseMorphs.push(mesh);
            });
        });
        const reel = new THREE.Group();
        const reelMaterial = new THREE.MeshStandardMaterial({
            color: 0xd4a017,
            metalness: 0.65,
            roughness: 0.3,
        });
        reel.add(new THREE.Mesh(new THREE.TorusGeometry(55, 10, 12, 32), reelMaterial));
        reel.add(
            new THREE.Mesh(
                new THREE.CylinderGeometry(12, 12, 8, 24),
                reelMaterial,
            ),
        );

        for (let index = 0; index < 5; index += 1) {
            const spoke = new THREE.Mesh(
                new THREE.BoxGeometry(12, 80, 8),
                reelMaterial,
            );
            spoke.rotation.z = (index * Math.PI) / 5;
            reel.add(spoke);
        }

        reel.position.set(-500, FLOOR + 270, 160);
        reel.userData.baseY = reel.position.y;
        reel.userData.phase = 0;
        reel.castShadow = true;
        scene.add(reel);
        expressiveObjects.push(reel);
        // === END TODO-2 ===

        // Handle resize
        window.addEventListener("resize", () =>
            onWindowResize(
                camera,
                renderer,
                window.innerWidth,
                window.innerHeight,
            ),
        );

        renderer.setAnimationLoop(animate);
    }

    // === TODO-3: Unit Visualization ===
    function createGenreGeometry(genre: string): THREE.BufferGeometry {
        switch (genre) {
            case "Comedy":
                return new THREE.SphereGeometry(6, 16, 12);
            case "Romance":
                return new THREE.ConeGeometry(7, 12, 6);
            case "Drama":
                return new THREE.CylinderGeometry(6, 6, 12, 8);
            default:
                return new THREE.BoxGeometry(10, 10, 10);
        }
    }

    function createGenreMaterial(genre: string): THREE.Material {
        return new THREE.MeshPhysicalMaterial({
            color: genreColors[genre],
            roughness: 0.45,
            metalness: 0.1,
            clearcoat: 0.25,
        });
    }

    function createUnits(scene: THREE.Scene, font: Font) {
        if (stackData.length === 0) return;

        const chartWidth = Math.min(stackData.length * 80, 1400);
        const unitHeight = 14;
        const xScale = d3
            .scaleBand()
            .domain(stackData.map((row) => row.year))
            .range([-chartWidth / 2, chartWidth / 2])
            .padding(0.1);
        const bandwidth = xScale.bandwidth();
        const unitScale = Math.min(1, bandwidth / 16);

        stackData.forEach((row, rowIndex) => {
            const xPosition = xScale(row.year)! + bandwidth / 2;
            let currentY = FLOOR;

            genres.forEach((genre) => {
                for (let index = 0; index < row[genre]; index += 1) {
                    const unit = new THREE.Mesh(
                        createGenreGeometry(genre),
                        createGenreMaterial(genre),
                    );
                    unit.scale.setScalar(unitScale);
                    unit.position.set(xPosition, currentY + unitHeight / 2, 0);
                    unit.castShadow = true;
                    unit.receiveShadow = true;
                    unit.userData.genre = genre;
                    scene.add(unit);
                    currentY += unitHeight;
                }
            });

            const numericYear = Number(row.year);
            const shouldShowYear =
                rowIndex === 0 ||
                rowIndex === stackData.length - 1 ||
                numericYear % 5 === 0;
            if (!shouldShowYear) return;

            const yearGeometry = new TextGeometry(row.year, {
                font,
                size: 6,
                depth: 2,
            });
            yearGeometry.computeBoundingBox();
            const yearWidth =
                yearGeometry.boundingBox!.max.x - yearGeometry.boundingBox!.min.x;
            const yearLabel = new THREE.Mesh(
                yearGeometry,
                new THREE.MeshPhysicalMaterial({ color: 0xffffff }),
            );
            yearLabel.position.set(
                xPosition - yearWidth / 2,
                FLOOR + 5,
                12,
            );
            yearLabel.castShadow = true;
            scene.add(yearLabel);
        });
    }

    function createLegend(scene: THREE.Scene, font: Font) {
        const legendX = 180;
        const legendStartY = FLOOR + 420;
        const spacing = 40;

        genres.forEach((genre, index) => {
            const yPosition = legendStartY - index * spacing;
            const swatch = new THREE.Mesh(
                createGenreGeometry(genre),
                createGenreMaterial(genre),
            );
            swatch.scale.setScalar(1.2);
            swatch.position.set(legendX, yPosition, 0);
            swatch.castShadow = true;
            scene.add(swatch);

            const label = new THREE.Mesh(
                new TextGeometry(genre, {
                    font,
                    size: 10,
                    depth: 3,
                }),
                new THREE.MeshPhysicalMaterial({ color: 0x000000 }),
            );
            label.position.set(legendX + 18, yPosition - 5, 0);
            label.castShadow = true;
            scene.add(label);
        });
    }

    function animate() {
        const delta = clock.getDelta();

        if (animating) {
            expressiveObjects.forEach((object) => {
                object.rotation.z += delta * 1.2;
                object.position.y =
                    object.userData.baseY +
                    Math.sin(clock.elapsedTime * 2 + object.userData.phase) * 12;
            });
            mixer.update(delta);
            horseMixer.update(delta);

            // TODO: Customize the animation behavior for your own models.
            // Birds - linear motion
            morphs.forEach((morph) => {
                morph.position.x += morph.speed * delta;
                if (morph.position.x > window.innerWidth / 2) {
                    morph.position.x = -window.innerWidth / 2 - Math.random() * 200;
                }
            });

            // TODO: Customize the animation for your own models.
            // Horses - orbit around the data
            horseMorphs.forEach((horse) => {
                horse.userData.angle += horse.userData.orbitSpeed * delta;
                const angle = horse.userData.angle;
                horse.position.x = ORBIT_RX * Math.cos(angle);
                horse.position.z = ORBIT_RZ * Math.sin(angle);

                // Face the direction of travel
                const lookAngle = angle + 0.1;
                horse.lookAt(
                    ORBIT_RX * Math.cos(lookAngle),
                    horse.position.y,
                    ORBIT_RZ * Math.sin(lookAngle),
                );
            });
        }

        renderer.render(scene, camera);
    }
</script>

<div bind:this={container} class="container"></div>

<button class="toggle-btn" onclick={() => animating = !animating}>
    {animating ? "Pause" : "Play"} Animation
</button>

<style>
    div.container {
        width: 100%;
        height: 100%;
    }

    .toggle-btn {
        position: fixed;
        bottom: 10px;
        right: 10px;
        padding: 8px 10px;
        font-size: 12px;
        border: none;
        border-radius: 20px;
        background: rgba(0, 0, 0, 0.30);
        color: white;
        cursor: pointer;
        z-index: 10;
    }

    .toggle-btn:hover {
        background: rgba(0, 0, 0, 0.8);
    }
</style>
