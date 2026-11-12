Degree level: PhD

# Project proposal

**Project title: An AI Agent for Efficient Object Onboarding in Model-Free 6-DoF Pose Estimation**

## 1. The problem and goal

**The problem and goal in my project:**

An object's **6-DoF pose** is its position (x, y, z) and orientation (roll, pitch, yaw) relative to the camera. A robot picking up a mug, for example, needs to know where the mug is and which way its handle points.

![A toy duck in a cluttered scene, highlighted in green, with three arrows showing its estimated orientation.](assets/bop-pose-example.png)

*Figure 1. 6-DoF pose estimation: the coordinate frame marks the duck's estimated position and orientation in the scene. Image: [BOP benchmark](https://bop.felk.cvut.cz/home/), Linemod (LM) dataset (Hinterstoisser et al., 2012).*

The pose-estimation system used in this project is **model-free**: it needs no CAD model (digital 3D design) of the object. Instead, a new object is **onboarded** using **reference images**, photos taken from different viewpoints, for example frames of a short video around it. A **3D reconstructor** combines the references and the available geometric information into a **point cloud**, a set of 3D points on the object's surface, which the estimator then uses to find the object's pose in new images.

Onboarding often provides more candidate references than are useful: neighbouring frames are nearly identical, some are blurry, and others repeat directions already covered. Using more references increases processing cost, but choosing a small set is not simply a matter of keeping the most different images. The references must expose useful surfaces while sharing enough information to be reconstructed together reliably. A new viewpoint may improve coverage but be poorly connected to the other views; several similar images may connect well while adding little new information.

![Five photos of the same toy duck on a cluttered table, taken from slightly different directions.](assets/bop-duck-references.png)

*Figure 2. Five reference photos of the duck from different viewpoints. Several are nearly identical. Given a larger candidate pool and a fixed reference budget, the agent must decide which photos are most useful together. Images: [BOP benchmark](https://bop.felk.cvut.cz/home/), Linemod (LM) dataset (Hinterstoisser et al., 2012).*

**Research motivation:** Conventional pose-estimation research commonly fixes the reference inputs and improves the pose-estimation model. This project investigates the complementary direction: **given a fixed tool, improve the references supplied to it during onboarding**. The question is whether an agent can use reconstruction feedback to choose a more useful reference set, improving downstream pose accuracy without changing the reconstruction or pose-estimation model.

**Goal and evaluation:** From a given pool, select useful references under a specified reference budget. Evaluation will use an **incremental-reference benchmark: BOP Average Recall (BOP-AR) versus the number of references**. BOP-AR measures pose-estimation accuracy on separate query images. For each predefined reference count *k*, the agent and baseline selectors must choose exactly *k* references from the same pool. **Success at a fixed count means higher BOP-AR than baseline selection using that same count.** Results will be reported separately at every count, as well as plotted together. This reveals how selection quality interacts with the reference budget, including whether its benefit diminishes as more references become available and whether better selection achieves comparable accuracy with fewer references.

The comparisons will include random selection, uniform frame sampling when the pool is an ordered sequence, and a greedy connectivity-based selector. The reconstruction program, pose estimator, candidate pools, and evaluation images remain the same across methods. Selection time, reconstruction calls, and requests for user assistance will also be recorded to show the cost of achieving any accuracy improvement. Methods that search using reconstruction feedback will receive the same reconstruction-call limit.

**Requirements:** The agent may select only from the supplied pool and must retain any mandatory references required by the input protocol; these count toward *k* and are shared across methods. It receives only the camera and depth information provided by that protocol. The pipeline and its settings remain fixed. Held-out query images and ground-truth poses are reserved for final evaluation: **BOP-AR does not feed back into the agent's selection loop**. Each run has a predefined time and reconstruction-call budget, and the user approves the final point cloud.

## 2. The simulator

**The simulator in my project:**

The program used inside the agent's loop is the **3D reconstructor** in an existing pose-estimation pipeline. It serves as the simulator for testing candidate selections: given a reference subset, it calculates the resulting point cloud and the connections between its observations. Both the reconstructor and the downstream pose estimator stay fixed. The pipeline runs locally on a GPU. Each reconstruction takes seconds, making repeated experiments practical. Seeds will be fixed where supported, and selections, settings, and outputs will be logged for reproducibility.

- **Inputs:** A text manifest, such as JSON or YAML, listing the selected reference-image paths and their available camera and depth information. The agent's task configuration specifies the candidate pool, target reference count, mandatory references, and experiment budget.
- **Outputs to the agent:** The coloured point cloud and its connectivity: which references observe each 3D point. A tool renders images of the point cloud from a few fixed angles for the agent to inspect alongside the reference photos. Reconstruction failures are also returned.
- **Separate evaluation:** After selection and user approval, the fixed estimator is evaluated on held-out query images to obtain BOP-AR. These scores measure the selection strategy's downstream value but are not available to guide its decisions.

## 3. The agent's role

**The agent in my project:**

The language-model-based agent runs a select, run, read, adjust loop for each target reference count:

1. Ask the user for any missing information about the reference pool, available camera information, intended viewing conditions, target reference count, and time limits.
2. Select an initial subset of exactly *k* references, seeking complementary views with enough overlap to connect them. Run the reconstructor.
3. Read the connectivity and inspect the rendered images of the coloured point cloud next to the reference photos.
4. Identify possible problems and test changes to the subset. The agent checks whether the shape and colours agree with the photos, whether background has been mixed in, whether a part appears twice or in the wrong place, and whether relevant surfaces are missing. It uses connectivity to trace suspicious regions to their supporting references, then tests replacements: remove a redundant or potentially problematic reference and add a candidate that may improve coverage or consistency. Connectivity suggests which references to investigate; the next reconstruction tests whether the replacement helps. If the evidence is ambiguous, the agent shows the user the images and asks a focused question.
5. Stop when further tested changes provide no useful improvement according to the reconstruction evidence, or when the experiment budget runs out. Present the final set of exactly *k* references and its point cloud for user approval, then export them or report unresolved problems.

![The agent's loop: the user sets goals and provides the reference pool; the language model selects a subset; the reconstructor returns a point cloud and its connectivity; the model reads them, then adjusts the selection, asking the user to judge the cloud when unsure; the user accepts the final point cloud, then the fixed estimator is evaluated on held-out images.](assets/approach-c-loop.svg)

*Figure 3. The agent's select, run, read, reason-and-adjust loop. For each benchmark run, the user specifies a reference count, and adding or dropping references is performed as a replacement so the selected count remains fixed. Step 02 uses the fixed reconstructor and returns point-cloud outputs, not pose-estimation scores. After the user approves the final cloud, the fixed estimator measures accuracy on held-out images; these scores do not feed back into the agent. The point cloud and the links between references under step 03 are illustrative. Duck images: BOP benchmark, Linemod (LM) dataset.*

**Variables and allowed ranges:** For a pool of *N* references, the agent chooses *k* distinct references, including any mandatory inputs. The benchmark specifies the tested counts, from one reference up to the whole pool where supported by the fixed tool. The agent changes which references are included and which replacements to test, within the experiment budget. It does not change the reconstruction algorithm, pose estimator, camera information, or model settings. Each reference count is evaluated as its own selection problem.

**What should the language model contribute?** A simple program can apply connectivity rules, for example preferring references that contribute points not already supported by others. This is why a greedy selector is a baseline. Connectivity alone, however, does not establish whether the coloured point cloud makes sense: a misplaced view may still share many points while producing a doubled beak, and background points may be well connected yet clearly not belong to the object. The hypothesis is that a language model with image understanding can combine visual evidence with connectivity to identify such problems, propose targeted replacements, and decide when to ask the user. The benchmark will test whether this reasoning improves downstream accuracy beyond simpler selection rules, rather than assuming that a visually plausible reconstruction guarantees better poses. User interventions will be logged so their contribution remains visible.

**One question I still need to answer:** Can the language model judge whether a coloured point cloud makes sense reliably enough to improve reference selection over a connectivity-based program at the same reference count, while needing user assistance only in ambiguous cases?

## Explain your project to someone outside your field

**My project in simple words:**

A robot picking up a mug must know where the mug is and which way its handle points. Software can learn a new object from photos taken around it: it combines them into a rough 3D shape made of points, then uses that shape to find the object in new camera pictures. A short video gives many photos, but some repeat the same view and others are difficult to combine. Researchers usually try to improve the recognition software. Here, I will keep the software fixed and investigate whether an AI assistant can make it work better by choosing its input photos more carefully.

For a given allowance of photos, the assistant will choose a set, build the 3D shape, and inspect the result alongside the original pictures. It will test replacing photos when the shape has gaps, duplicated parts, or unwanted background. When it cannot judge the result confidently, it will ask a person, who also approves the final shape. I will compare its selections with simpler methods using exactly the same number of photos, and repeat this for several photo allowances. Accuracy on separate test pictures will show which method chooses more useful photos at each allowance, while the time spent selecting them will show the cost of that improvement.
