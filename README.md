import simpy
import geopy.distance
import matplotlib.pyplot as plt
import folium
import numpy as np
import random
import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import webbrowser


class Highway:
    def __init__(self, start, end):
        self.start = start
        self.end = end
        self.length_km = geopy.distance.distance(start, end).km

    def interpolate(self, fraction):
        start_lat, start_lon = self.start
        end_lat, end_lon = self.end
        interpolated_lat = start_lat + fraction * (end_lat - start_lat)
        interpolated_lon = start_lon + fraction * (end_lon - start_lon)
        return (interpolated_lat, interpolated_lon)


class TollPoint:
    def __init__(self, location, name):
        self.location = location
        self.name = name


class Car:
    def __init__(self, id, start_location, destination, speed):
        self.id = id
        self.start_location = start_location
        self.destination = destination
        self.speed = speed


def calculate_distance(point1, point2):
    return geopy.distance.distance(point1, point2).km


def car_process(env, car, highway, toll_points, toll_rate_per_km, car_positions, entered_toll_zones, toll_events):
    print(f"Car {car.id} starts at {car.start_location} with speed {car.speed} km/h")

    total_distance = calculate_distance(car.start_location, car.destination)
    distance_covered = 0
    time_step = 1
    total_toll_amount = 0

    while distance_covered < total_distance:
        yield env.timeout(time_step)
        distance_covered += car.speed * time_step
        current_position = highway.interpolate(distance_covered / total_distance)
        car_positions.append(current_position)

        for toll in toll_points:
            if calculate_distance(current_position, toll.location) < (car.speed * time_step):
                if toll not in entered_toll_zones and random.random() < 0.5:
                    entered_toll_zones.append(toll)
                    toll_events.append((env.now, toll.location, "enter"))
                    print(f"Car {car.id} enters toll zone at {toll.name}")
                    # Calculate toll amount for this segment
                    segment_distance = calculate_distance(car.start_location, current_position)
                    toll_amount = segment_distance * toll_rate_per_km
                    total_toll_amount += toll_amount

        print(f"Car {car.id} is at {current_position}")

    toll_events.append((env.now, car.destination, "exit"))
    print(f"Car {car.id} reaches destination {car.destination}")

    # Return the total toll amount
    return total_toll_amount


coimbatore = (11.0168, 76.9558)
bangalore = (12.9716, 77.5946)

# Initialize the highway
highway = Highway(coimbatore, bangalore)

# Define three toll points along the highway
toll_points = [
    TollPoint(highway.interpolate(0.3), "Toll Point 1"),  # Toll point at 30% of the way
    TollPoint(highway.interpolate(0.5), "Toll Point 2"),  # Toll point at 50% of the way
    TollPoint(highway.interpolate(0.7), "Toll Point 3")   # Toll point at 70% of the way
]

# Initialize one car with specific start and destination coordinates and speed
start_location = coimbatore
destination = bangalore
car_speed = random.uniform(50, 100)
car = Car(1, start_location, destination, car_speed)

env = simpy.Environment()

# Define toll rate per km
toll_rate_per_km = 0.25  # 0.0025 INR per km

car_positions = [start_location]
entered_toll_zones = []
toll_events = []
total_toll = 0


def run_simulation():
    global total_toll
    total_toll = env.process(car_process(env, car, highway, toll_points, toll_rate_per_km, car_positions, entered_toll_zones, toll_events))
    env.run()


def start_simulation():
    run_simulation()
    messagebox.showinfo("Simulation", "Simulation started")


def end_simulation():
    messagebox.showinfo("Simulation", f"Total Toll Amount: {total_toll.value:.2f} INR")
    plot_results()


def plot_results():
    total_distance = calculate_distance(car.start_location, car.destination)
    distance_in_toll_zones = 0

    for toll in entered_toll_zones:
        if calculate_distance(car.start_location, toll.location) < total_distance:
            distance_to_toll = calculate_distance(car.start_location, toll.location)
            distance_from_toll = calculate_distance(toll.location, car.destination)
            distance_in_toll_zones += min(distance_to_toll, distance_from_toll, total_distance)

    print(f"\nSimulation results:")
    print(f"Car {car.id}: Start: {car.start_location}, Destination: {car.destination}")
    print(f"Total Distance: {total_distance:.2f} km, Distance in Toll Zones: {distance_in_toll_zones:.2f} km, Speed: {car.speed:.2f} km/h")

    plt.clf()

    fraction_steps = np.linspace(0, 1, num=100)
    highway_points = [highway.interpolate(f) for f in fraction_steps]

    highway_lats, highway_lons = zip(*highway_points)
    car_lats, car_lons = zip(*car_positions)

    plt.plot(highway_lons, highway_lats, label='Highway', color='blue')

    in_toll_zone = False
    segment_start = car_positions[0]
    for i in range(1, len(car_positions)):
        segment_end = car_positions[i]
        for toll in toll_events:
            if segment_start == toll[1] and toll[2] == 'enter':
                in_toll_zone = True
            elif segment_end == toll[1] and toll[2] == 'exit':
                in_toll_zone = False

        color = 'red' if in_toll_zone else 'white'
        plt.plot([segment_start[1], segment_end[1]], [segment_start[0], segment_end[0]], color=color)
        segment_start = segment_end

    plt.scatter([toll.location[1] for toll in toll_points], [toll.location[0] for toll in toll_points], marker='x', label='Toll Points', color='green')
    plt.scatter(coimbatore[1], coimbatore[0], label='Start: Coimbatore', color='orange')
    plt.scatter(bangalore[1], bangalore[0], label='Destination: Bangalore', color='purple')

    for event in toll_events:
        event_time, event_location, event_type = event
        plt.scatter(event_location[1], event_location[0], marker='o', label=f'{event_type.title()} Toll at {event_time}', color='black' if event_type == 'enter' else 'yellow')

    plt.title('Car Path on the Highway with Toll Points')
    plt.xlabel('Distance')
    plt.ylabel('Time')
    plt.legend()
    plt.grid(True)
    plt.show()

    display_folium_map()


def display_folium_map():
    mymap = folium.Map(location=(11.5296, 78.7742), zoom_start=7)
    folium.PolyLine([highway.start, highway.end], color='blue', weight=2.5, opacity=1).add_to(mymap)

    in_toll_zone = False
    segment_start = car_positions[0]
    for i in range(1, len(car_positions)):
        segment_end = car_positions[i]
        for toll in toll_events:
            if segment_start == toll[1] and toll[2] == 'enter':
                in_toll_zone = True
            elif segment_end == toll[1] and toll[2] == 'exit':
                in_toll_zone = False

        color = 'red' if in_toll_zone else 'blue'
        folium.PolyLine([segment_start, segment_end], color=color, weight=5, opacity=1).add_to(mymap)
        segment_start = segment_end

    folium.Marker(coimbatore, popup='Start: Coimbatore', icon=folium.Icon(color='orange')).add_to(mymap)
    folium.Marker(bangalore, popup='Destination: Bangalore', icon=folium.Icon(color='purple')).add_to(mymap)
    for i, toll in enumerate(toll_points):
        folium.Marker(toll.location, popup=f'{toll.name}', icon=folium.Icon(color='green')).add_to(mymap)

    current_position = car_positions[-1]
    for event in toll_events:
        if event[2] == 'enter' and event[0] <= env.now:
            current_position = event[1]
    folium.Marker(current_position, popup='Current Position', icon=folium.Icon(color='red')).add_to(mymap)

    for event in toll_events:
        event_time, event_location, event_type = event
        folium.Marker(event_location, popup=f'{event_type.title()} Toll at {event_time}', icon=folium.Icon(color='black' if event_type == 'enter' else 'yellow')).add_to(mymap)

    mymap.save('car_path_map.html')
    webbrowser.open('car_path_map.html')


def show_receipt():
    receipt_window = tk.Toplevel(root)
    receipt_window.title("Payment Receipt")
    receipt_window.configure(bg='#87CEEB')  # Sky blue

    receipt_label = tk.Label(receipt_window, text="Payment Receipt", font=("Helvetica", 18), bg='#87CEEB')
    receipt_label.pack(pady=10)

    current_datetime = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    transaction_id = random.randint(1000000000, 9999999999)
    payment_mode = 'Credit Card'

    details = f"Amount Paid: {total_toll.value:.2f} INR\n"
    details += f"Date and Time: {current_datetime}\n"
    details += f"Vehicle Type: Car\n"
    details += f"Speed: {car.speed:.2f} km/h\n"
    details += f"Transaction ID: {transaction_id}\n"
    details += f"Payment Mode: {payment_mode}"

    receipt_details = tk.Label(receipt_window, text=details, font=("Helvetica", 14), bg='#87CEEB')
    receipt_details.pack(pady=10)

    close_btn = tk.Button(receipt_window, text="Close", command=receipt_window.destroy, font=("Helvetica", 14), width=10)
    close_btn.pack(pady=10)


root = tk.Tk()
root.title("GPS Toll Simulation")
root.configure(bg='#87CEEB')

container = tk.Frame(root, bg='#87CEEB')
container.pack()

title_label = tk.Label(container, text="GPS Toll Simulation", font=("Helvetica", 24), bg='#87CEEB')  # Sky blue
title_label.pack(pady=20)

box = tk.Frame(root, bg='#87CEEB')
box.pack()

start_btn = tk.Button(box, text="Start Simulation", command=start_simulation, font=("Helvetica", 16), width=20, height=2)
start_btn.pack(side=tk.LEFT, padx=20, pady=50)

end_btn = tk.Button(box, text="End Simulation", command=end_simulation, font=("Helvetica", 16), width=20, height=2)
end_btn.pack(side=tk.LEFT, padx=20, pady=50)

receipt_btn = tk.Button(box, text="Toll Bill", command=show_receipt, font=("Helvetica", 16), width=20, height=2)
receipt_btn.pack(side=tk.LEFT, padx=20, pady=50)

footer = tk.Frame(root, bg='#87CEEB')
footer.pack(pady=50)

footer_label = tk.Label(footer, text="GPS TOLL BASED SIMULATION USING PYTHON", font=("Helvetica", 24), bg='#87CEEB')  # Sky blue
footer_label.pack()

root.mainloop()
