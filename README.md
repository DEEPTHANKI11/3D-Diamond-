import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d.art3d import Poly3DCollection
from tkinter import Tk, Scale, Button, HORIZONTAL, Frame, Label
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg, NavigationToolbar2Tk
import matplotlib
from tkinter.colorchooser import askcolor

matplotlib.use('TkAgg')

# Default color for the diamond
diamond_color = 'red,blue'

# Initial rotation angles
azimuth_angle = 0
elevation_angle = 30

# Global variables
zoom_level = 2
light_on = True
refresh_rate = 100  # milliseconds

def generate_diamond_with_side_and_angled_cuts(height, weight):
    # Parameters for the diamond
    upper_cut_radius = 0.5 * weight
    lower_cut_radius = 0.5 * weight
    upper_cut_height = 0.5
    lower_cut_height = 0.5
    girdle_radius_top = 1.01 * weight
    girdle_radius_bottom = 0.9 * weight
    girdle_height = 0.1

    num_top_facets = 8
    num_bottom_facets = 16

    angles_top = np.linspace(0, 2 * np.pi, num_top_facets, endpoint=False)
    angles_bottom = np.linspace(0, 2 * np.pi, num_bottom_facets, endpoint=False)

    top_vertex = np.array([0, 0, height / 2])
    bottom_vertex = np.array([0, 0, -height / 2])

    x_upper = upper_cut_radius * np.cos(angles_top)
    y_upper = upper_cut_radius * np.sin(angles_top)
    z_upper = np.full(num_top_facets, height / 2 - upper_cut_height)
    upper_cut_vertices = np.vstack([x_upper, y_upper, z_upper]).T

    x_girdle_bone = girdle_radius_top * np.cos(angles_top)
    y_girdle_bone = girdle_radius_top * np.sin(angles_top)
    z_girdle_bone = np.full(num_top_facets, girdle_height / 2)
    girdle_bone_vertices = np.vstack([x_girdle_bone, y_girdle_bone, z_girdle_bone]).T

    x_girdle_valley = girdle_radius_bottom * np.cos(angles_bottom)
    y_girdle_valley = girdle_radius_bottom * np.sin(angles_bottom)
    z_girdle_valley = np.full(num_bottom_facets, -girdle_height / 2)
    girdle_valley_vertices = np.vstack([x_girdle_valley, y_girdle_valley, z_girdle_valley]).T

    x_lower = lower_cut_radius * np.cos(angles_bottom)
    y_lower = lower_cut_radius * np.sin(angles_bottom)
    z_lower = np.full(num_bottom_facets, -height / 2 + lower_cut_height)
    lower_cut_vertices = np.vstack([x_lower, y_lower, z_lower]).T

    vertices = np.vstack([
        top_vertex, 
        upper_cut_vertices, 
        girdle_bone_vertices,
        girdle_valley_vertices,
        lower_cut_vertices, 
        bottom_vertex
    ])

    faces = []

    for i in range(num_top_facets):
        next_i = (i + 1) % num_top_facets
        faces.append([5, i + 1, next_i + 1])

    for i in range(num_top_facets):
        next_i = (i + 1) % num_top_facets
        faces.append([i + 1, next_i + 1, num_top_facets + next_i + 1, num_top_facets + i + 1])

    for i in range(num_top_facets):
        next_i = (i + 1) % num_top_facets
        corresponding_valley_i = int(i * (num_bottom_facets / num_top_facets))
        corresponding_valley_next_i = int(next_i * (num_bottom_facets / num_top_facets))
        faces.append([num_top_facets + i + 1, num_top_facets + next_i + 1, 
                      2 * num_top_facets + corresponding_valley_next_i + 1, 
                      2 * num_top_facets + corresponding_valley_i + 1])

    for i in range(num_bottom_facets):
        next_i = (i + 1) % num_bottom_facets
        faces.append([2 * num_top_facets + i + 1, 2 * num_top_facets + next_i + 1, len(vertices) - 1])

    return vertices, faces

def plot_diamond(height, weight):
    vertices, faces = generate_diamond_with_side_and_angled_cuts(height, weight)
    
    ax.clear()
    
    # Define a color map with alpha transparency for a glass effect
    num_faces = len(faces)
    colors = plt.cm.viridis(np.linspace(0, 1, num_faces))
    colors[:, 3] = 0.5  # Adjust transparency for glass-like effect
    
    poly3d = [[vertices[vert_id] for vert_id in face] for face in faces]
    collections = [Poly3DCollection([poly3d[i]], facecolors=colors[i], linewidths=0.5, edgecolors='black', alpha=0.6) for i in range(num_faces)]
    
    for collection in collections:
        ax.add_collection3d(collection)
    
    scale = vertices.flatten()
    ax.auto_scale_xyz(scale, scale, scale)
    
    ax.view_init(elevation_angle, azimuth_angle)
    
    ax.set_xlim([-zoom_level, zoom_level])
    ax.set_ylim([-zoom_level, zoom_level])
    ax.set_zlim([-zoom_level, zoom_level])
    ax.set_facecolor('white')  # Set background color to white
    ax.axis('off')
    canvas.draw()

def update_plot():
    height = w1.get() / 10
    weight = w2.get() / 10
    plot_diamond(height, weight)

def zoom_in():
    global zoom_level
    zoom_level /= 1.2
    update_plot()

def zoom_out():
    global zoom_level
    zoom_level *= 1.2
    update_plot()

def rotate_left():
    global azimuth_angle
    azimuth_angle -= 10
    update_plot()

def rotate_right():
    global azimuth_angle
    azimuth_angle += 10
    update_plot()

def reset_view():
    global azimuth_angle, elevation_angle
    azimuth_angle = 0
    elevation_angle = 30
    update_plot()

def change_color():
    global diamond_color
    color = askcolor()[1]
    if color:
        diamond_color = color
        update_plot()

def toggle_light():
    global light_on
    light_on = not light_on
    if light_on:
        light_button.config(text='Turn Light Off')
    else:
        light_button.config(text='Turn Light On')
    update_plot()

def increase_refresh_rate():
    global refresh_rate
    refresh_rate = max(10, refresh_rate - 10)  # Ensure a minimum refresh rate
    refresh_rate_label.config(text=f"Refresh Rate: {refresh_rate} ms")
    schedule_plot_update()

def decrease_refresh_rate():
    global refresh_rate
    refresh_rate += 10
    refresh_rate_label.config(text=f"Refresh Rate: {refresh_rate} ms")
    schedule_plot_update()

def schedule_plot_update():
    master.after(refresh_rate, update_plot)  # Schedule update_plot function

master = Tk()
master.title("DEEP THANKI")

frame = Frame(master)
frame.pack(pady=10, fill='x', expand=True)

w1 = Scale(master, from_=10, to=50, label="Height", orient=HORIZONTAL)
w1.set(19)
w1.pack(fill='x', padx=10)

w2 = Scale(master, from_=10, to=50, label="Weight", orient=HORIZONTAL)
w2.set(10)
w2.pack(fill='x', padx=10)

button_frame = Frame(master)
button_frame.pack(pady=10, fill='x', expand=True)

Button(button_frame, text='Zoom In', command=zoom_in).pack(side='left', padx=5)
Button(button_frame, text='Zoom Out', command=zoom_out).pack(side='left', padx=5)
Button(button_frame, text='Rotate Left', command=rotate_left).pack(side='left', padx=5)
Button(button_frame, text='Rotate Right', command=rotate_right).pack(side='left', padx=5)
Button(button_frame, text='Reset View', command=reset_view).pack(side='left', padx=5)
Button(button_frame, text='Change Color', command=change_color).pack(side='left', padx=5)

light_button = Button(master, text='Turn Light Off', command=toggle_light)
light_button.pack(pady=10)

refresh_frame = Frame(master)
refresh_frame.pack(pady=10, fill='x', expand=True)

Button(refresh_frame, text='Increase Refresh Rate', command=increase_refresh_rate).pack(side='left', padx=5)
Button(refresh_frame, text='Decrease Refresh Rate', command=decrease_refresh_rate).pack(side='left', padx=5)

refresh_rate_label = Label(refresh_frame, text=f"Refresh Rate: {refresh_rate} ms")
refresh_rate_label.pack(side='left', padx=5)

fig = plt.figure(figsize=(8, 8))
fig.patch.set_facecolor('white')
ax = fig.add_subplot(111, projection='3d')

canvas = FigureCanvasTkAgg(fig, master=master)
canvas.get_tk_widget().pack(fill='both', expand=True)
toolbar = NavigationToolbar2Tk(canvas, master)
toolbar.update()
canvas.get_tk_widget().pack(fill='both', expand=True)

plot_diamond(1.9, 1.0)
schedule_plot_update()  # Ensure periodic updates

master.mainloop()
