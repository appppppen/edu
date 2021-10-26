### NodeMaterial with GlowLayer Demo

```typescript
import {
    Vector3, Engine, Scene, Animation, Color3, EasingFunction, SceneLoader,
    ArcRotateCamera, NodeMaterial, GlowLayer, SineEase, TextureBlock, InputBlock
} from '@babylonjs/core/Legacy/legacy';
import '@babylonjs/loaders';
import { BaseScene } from '../../../../babylon/template/Base/BaseScene';
import * as lightFixture from '../sub_static/lightFixture.glb';
import * as lightGlowMat from '../sub_static/lightGlowMat.json';
export class AssembleScene extends BaseScene {

    constructor() {
        super();
        this.init();
    }

    /* 创建场景*/
    createScene(engine: Engine): Scene {
        const canvas = engine.getRenderingCanvas();
        const scene = new Scene(engine);
        scene.clearColor.set(0.2, 0.2, 0.2, 1);
        const camera = new ArcRotateCamera('cam', 1.2, Math.PI / 2.5, 3, Vector3.Zero(), scene);
        camera.wheelDeltaPercentage = 0.01;
        camera.attachControl(canvas, true);

        // new node material for light animation
        const lightNodeMat = new NodeMaterial('lightNodeMat', scene, { emitComments: false });

        // meshes and shaders to load
        const promises = [];
        promises.push(SceneLoader.AppendAsync(lightFixture));
        // promises.push(lightNodeMat.loadAsync(lightGlowMat));
        promises.push(lightNodeMat.loadFromSerialization(lightGlowMat));

        // load all and finish scene when assets loaded
        Promise.all(promises).then(function () {

            // create a camera pointing at your model.
            scene.createDefaultLight();

            // create default environment
            const helper = scene.createDefaultEnvironment();
            helper.setMainColor(Color3.Gray());

            // get light mesh, material, and textures
            const lightMesh = scene.getMeshByName('lightTube');
            const loadedTextures = lightMesh.material.getActiveTextures();
            let lightBaseColorTex;
            let lightEmissiveTex;

            for (let i = 0; i < loadedTextures.length; i++) {
                if (loadedTextures[i].name.includes('(Base Color)')) {
                    lightBaseColorTex = loadedTextures[i];
                } else if (loadedTextures[i].name.includes('(Emissive)')) {
                    lightEmissiveTex = loadedTextures[i];
                }
            }
            // build node material
            lightNodeMat.build(false);
            lightMesh.material = lightNodeMat;
            // assign original textures to node material
            const baseColor = lightNodeMat.getBlockByName('baseColorTexture') as TextureBlock;
            const emissiveColor = lightNodeMat.getBlockByName('emissiveTexture') as TextureBlock;

            baseColor.texture = lightBaseColorTex;
            emissiveColor.texture = lightEmissiveTex;

            // get shader values to drive glow
            const glowMask = lightNodeMat.getBlockByName('glowMask') as InputBlock;
            const emissiveStrength = lightNodeMat.getBlockByName('emissiveStrength') as InputBlock;

            // set up glow layer post effect
            const gl = new GlowLayer('glow', scene);
            gl.intensity = 1.25;

            // set up material to use glow layer
            gl.referenceMeshToUseItsOwnMaterial(lightMesh);

            // enable glow mask to render only emissive into glow layer, and then disable glow mask
            gl.onBeforeRenderMeshToEffect.add(() => {
                glowMask.value = 1.0;
            });
            gl.onAfterRenderMeshToEffect.add(() => {
                glowMask.value = 0.0;
            });

            // flicker animation
            const flickerAnim = new Animation('flickerAnim', 'value', 60, Animation.ANIMATIONTYPE_FLOAT, Animation.ANIMATIONLOOPMODE_CYCLE);
            const easingFunction = new SineEase();
            easingFunction.setEasingMode(EasingFunction.EASINGMODE_EASEINOUT);
            const flickerKeys = [
                { frame: 0, value: 0.2 },
                { frame: 5, value: 0.8 },
                { frame: 10, value: 0.1 },
                { frame: 25, value: 0.8 },
                { frame: 30, value: 0.05 },
                { frame: 35, value: 0.7 },
                { frame: 40, value: 0.3 },
                { frame: 55, value: 0.5 },
                { frame: 70, value: 0.35 },
                { frame: 170, value: 1.0 }
            ];
            flickerAnim.setKeys(flickerKeys);
            flickerAnim.setEasingFunction(easingFunction);
            scene.beginDirectAnimation(emissiveStrength, [flickerAnim], 0, flickerKeys[flickerKeys.length - 1].frame, true, 1);
        });
        return scene;
    }

}

```
