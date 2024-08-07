# multimodal

This is a collection of scripts in Julia to preprocess the multimodal naturalistic data collected using Lab Streaming Layer and Pupil Core mobile eye-tracker. `Functions.jl` contains a collection of Julia functions designed for processing eye-tracking data, handling various data formats, and performing transformations. These functions are particularly tailored for working with data from the DGAME project. The data comes in .csv files for fixations, gazes and audio annotations, .mp4 files for video, and .xdf, .json and binaty formats for other files.

The DGAME project is a naturalistic interactive experimental setting, where two participant separated by an obstacle (a 4x4 wooden shelf) have to reorder the objects on the shelf. Objects may be unique (one signgle batter per shelf) or duplicated (two identical candles). the Director has a stack of cards with pictures of the two adjacent cells of the shelf holding objects, then they have to come up with the instructions for the Matcher to move one of the objects to match the picture. Some of the cells are closed from the side of the Director, so they cannot see all the objects. In half of the trials the Director and the Matcher cannot see the faces of each other. Every pair of participants have four sessions, 10 minutes each.

Please see the 'sample data structure DGAME. txt' for the structure of folders and files from the eye-tracker and annotations. The description of the data structure is in 'data structure description.txt'

The object positions on the shelf annotations are done with computer vision (yolo)

The preprocessing is done in the following steps:
- Read all the Lab Streaming Layer timestamps from  .xdf files and aggregate them in one table
- Read all the eye-tracker timestamps from .json files from the eye-tracker raw data for the director and the matcher and aggregate them in one table
- Get all the frames of interest (200 milliseconds primary to the noun onset) and recognize object positions with pretrained yolo CV model.
- Get all coordinates for all recognized objects for all frames and write them to one dataset
- 
For each Director-Matcher pair, for each session:
 -   Read the audio transcription, select the target object mentions, tokenize all nouns (participant can call the same object with different words)
 -   Read all surface fixations files, clean by surface, combine into one dstaset, align timelines, filter by +- n milliseconds from target noun onset
 -   Get all the surface coordinates, perspective transform into picture pixel coordinates
 -   Get all the object coordinates from the preaggregated .csv
 -   Get all the gazes and fixations for the director and the Matcher: for target objects only, for all objects and the face of the other participant

In the last month the priority was to get everything to work, so I apologize for the lack of documentation. I am planning to get everything nicely documentd in the nearest time.

## Installation

To use the functions in this repository, ensure you have the following Julia packages installed:

```julia
using Pkg
Pkg.add("XDF")
Pkg.add("EzXML")
Pkg.add("XMLDict")
Pkg.add("JSON")
Pkg.add("LinearAlgebra")
Pkg.add("TextParse")
Pkg.add("MsgPack")
Pkg.add("CSV")
Pkg.add("DataFrames")
Pkg.add("CairoMakie")
Pkg.add("Images")
Pkg.add("Printf")
```

Run the pipeline: here is the script that runs the pipeline to get all fixations +- 1 sec from the noun onset, and separately target object fixations for the director and the matcher

```julia
# this is the main script, that runs the pipeline
include("functions.jl")
root_folder = ""
labels_folder = "labels"

# every set has two participants and four sessions
sets = ["04", "05", "06", "07", "08", "10", "11", "12"]
surface_sessions = Dict([("01", "000"), ("02", "001"), ("03", "002"), ("04", "003")])

#Read all the Lab Streaming Layer timestamps from .xdf and .json files and aggregate them in one table
get_all_timestamps_xdf(sets, root_folder)
get_all_timestamps_json(sets, root_folder)
get_lag_ET()
```
Get all the frames of interest (200 milliseconds primary to the noun onset):
check if all the april tags are recognized, if not, get a frame with maximum april tags from 1 sec to the noun onset period
get all the fixations for the time period of interest for all participants:
Read the audio transcription, select the target object mentions, tokenize all nouns (participant can call the same object with different words)
Read all surface fixations files, clean by surface, combine into one dstaset, align timelines, filter by +- n milliseconds from target noun onset
At the moment tjis is +- 1 sec from the word onset

```julia
# Open a log file for writing
log_file = open("combine_fixations_by_nouns.log", "w")
# Redirect stdout to the log file
redirect_stdout(log_file)
    @info "This is the log file of the processing of the fixations by nouns, you can find all the missing values and errors here"
all_trial_surfaces_gazes, all_trial_surfaces_fixations = get_all_gazes_and_fixations_by_frame(sets)
# Yolo may change image size deleting the black borders, so we need to check the image sizes
close(log_file)

# get frames of interest (200 ms before the noun onset)
frames = get_frames_from_fixations(all_fixations)
#correct frame numbers according to april tags recognized
#get a frame with maximum april tags from 1 sec to the noun onset period
frames_corrected = check_april_tags_for_frames(frames)

#read from file if needed, CSV package cannot handle surface transformation matrices, so use TextParse
frames_corrected = CSV.read("frame_numbers_corrected_with_tokens.csv", DataFrame)
if isempty(surface_positions)
    data, surf_names = TextParse.csvread("all_surface_matrices.csv")
    surface_positions =  DataFrame()
    for (i, surf_name) in enumerate(surf_names)
        surface_positions[!, Symbol(surf_name)] = data[i]
    end
end


#get all transformation matrices for all frames in one aggregated table
#it will be written to a cvs file "all_surface_matrices.csv"
surface_positions = get_all_surface_matrices_for_frames(frames_corrected)

#Get all coordinates for all recognized objects for all frames and write them to one dataset
#it will be written to a cvs file "all_yolo_coordinates.csv"
#lables_folder is a folder with labels .txt files for tne frames woth objects recognized by Yolo
yolo_coordinates = get_all_yolo_coordinates(labels_folder)
```

Read the audio transcription, select the target object mentions, tokenize all nouns (participant can call the same object with different words),
Read all surface fixations files, clean by surface, combine into one dstaset, align timelines, filter by +- n milliseconds from target noun onset
At the moment trial is +- 1 sec from the word onset
!NB Yolo may change image size deleting the black borders, so we need to check the image sizes

```julia
yolo_output_path = "path/to/yolo/output"
image_sizes = collect_image_dimensions(yolo_output_path)
```


We have aggregated yolo normalized coordinates for all objects recognized in the frames and saved in "all_yolo_coordinates.csv",
surface_positions is a table with all the surfaces and their positions that we have put into "all_surface_matrices.csv"

!NB Check your recognized image sizes and correct them if needed, otherwise the pixel coordinates will not be calculated properly
!NB not all frames have recognized surfaces, even if all the april tags are visible, it might be the case that there are no surfaces for 30 frames in a row (e.g. 08_01, session 3, frames 19763-1979, no surfaces recognized)
in this case object position will be 'outside all'

```julia
#get all surfaces for all recogized objects for every frame and write the aggregated table to a csv file
#it will be written to a cvs file "all_frame_objects.csv"
all_frame_objects = get_surfaces_for_all_objects(yolo_coordinates, surface_positions, root_folder, frames_corrected, image_sizes)

#Get all the fixations for target objects only 
#(for the object called in the current noun, 1 sec before and after noun onset)

target_gazes, target_fixations =  get_gazes_and_fixations_by_frame_and_surface(all_frame_objects, all_trial_surfaces_gazes, all_trial_surfaces_fixations)  

#if something is wrong with the coordinates, you can try plot surfaces to find out
# e.g.for this particular frame there are no surfaces provided by the Pupil Core plugin
surface_coordinates=get_all_surfaces_for_a_frame(19787, frame_surfaces)
plot_surfaces(surface_coordinates, img_width, img_height, "path/to/the/frame.jpg")

```
## Functions
Please see the function list.md for the full documentation