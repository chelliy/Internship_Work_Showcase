# PCG Blueprint Class Explanation Document

There are currently two completed PCG classes (Blueprint Class): one for the environment (PCG_Environment_Spline) and one for roads (PCG_Road_Spline).  
The environment class comes with a spline when dragged into the Editor. After selecting the static mesh object to be generated, the object will appear along the spline, and users can shape the environment by adding and dragging spline points.  
The road class works similarly to the environment class but with fewer features, which will be explained later.

The current development focus of PCG is on implementing the rule that allows the environment to avoid roads.

PCG has two folders: an old version and a current version, both found in the project’s **Content/PCGTool** directory.

The only difference between the two versions is how the environment avoids roads.
- The old version uses a brute-force method to make all environmental objects colliding with roads invisible. This version works properly.
- The current version finds the collision points between the road and environment, identifies the spline segments of the environment that cannot be used, and regenerates the environment accordingly. The basic framework and logic for the new version have been completed, but numerous bugs prevent it from functioning correctly.

## Detailed Explanation

### Environment Class (PCG_Environment_Spline)
The logic for generating environment objects is as follows: after placing an object at the current location on the spline, the next usable location is calculated, and the next object is placed. This cycle continues until no usable positions remain. On the spline, this is reflected by determining the position based on the **Distance** of the spline.

- **Auto Spacing Mode**: The program calculates the distance needed to connect two objects based on their size. However, due to the properties of the Spline Mesh Component, where start and end points must be provided for object generation, the implementation doesn’t fully follow this logic. This mode does not allow for rotation or scaling of generated objects. Currently, users can only select one object for generation.

- **Non-Auto Spacing Mode**: This mode allows users to specify the distance between objects, or a random distance within a specified range. Because it uses a different implementation (Spawn Static Mesh), users can apply uniform or random rotation and scaling to each generated mesh. Multiple objects can be selected for generation in this mode.

### Road Class (PCG_Road_Spline)
The road generation logic is the same as the environment class, but it only supports Auto Spacing mode.

## Environment Avoidance of Roads Rule (Old Version)
The idea is to hide any environmental meshes that overlap with roads.

Since roads are composed of connected spline meshes, this rule is implemented by recording all generated spline mesh components. Each mesh is checked for overlaps with environment meshes using the **component overlap components** method, and if overlaps are found, the environment mesh is set to invisible and recorded. If the road class is regenerated, any modified environment meshes are restored.

However, this method leaves noticeable gaps between the road and the remaining environment objects.

## Environment Avoidance of Roads Rule (New Version)
The idea is to move overlapping environment meshes to the edge of the road, solving the gap issue from the old version.

### Implementation
- All generated spline mesh components are recorded. When traversing these meshes, two boundaries are identified for each mesh (e.g., for a rectangular road mesh, the long sides of the rectangle are the boundaries). A **multi-line trace** is used to find all environmental objects colliding with these boundaries, and the collision points are recorded on the environment’s spline (storing **Distance** data).
- The collision points are then sorted by distance, and for each point, the system checks if the spline segment between the current and next point is still usable. This is done by taking the midpoint of the current segment and checking with a **line trace** in the Up Vector direction to see if it collides with a road object. If there is a collision, the segment is unusable; otherwise, it remains usable. Once all collision points are checked, the environment’s usable segment data is updated, and the environment is regenerated.

This method has been set up but is not yet functional due to several bugs.

### Bugs:
When dragging the environment class after initial generation (not tested for the initial position):
1. The environment returns to its original position after colliding with the road and regenerating.
2. The line generated by the midpoint detection method is not in the correct position and appears at the original generation location.
3. No noticeable gaps are left for the road after regeneration, suggesting that the update to the usable spline segments might have failed.

### Possible Causes:
- **For issues 1, 2, and 3**: There may be a problem with using **Attach Component to Component**. None of the PCG-generated objects are visible in the Editor’s display list, and although individual environment objects can be forcefully selected in the scene, they lack a Details panel. This could also prevent the environment class from updating the spline position in real-time.
- **For issue 3**: Misuse of **Structure**. Blueprints cannot directly generate nested array variables. A method learned online suggests using a custom structure to store an array, then creating an array of that structure in Blueprints. In the road class, a **Map** variable is created where the key is the environment class and the value is a custom structure (containing a Float Map storing unusable segment information, with the start point as the key and the endpoint as the value). Initial testing shows that the array in the structure is empty, which could indicate null references. However, it could also be caused by the previous issue, which prevents unusable segments from being added.