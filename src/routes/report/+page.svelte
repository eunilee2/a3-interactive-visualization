<script lang="ts">
    import type { PageProps } from './$types';

    let { data }: PageProps = $props();
</script>

<h1>Report Writeup</h1>
<a href="/part2">
	<button type="button">Go to Part 2 Chart &larr;</button>
</a>

<a href="/">
	<button type="button">Go back to the Part 1 Chart &rarr;</button>
</a>

<div class="report-content">
    <ul>
        <li>A link to the deployed project. Add a link to where the interactive visualization is hosted.</li>
        <p><a href="https://a3-interactive-visualization.vercel.app/">a3-interactive-visualization.vercel.app/</a></p>
        <li>A rationale for your design decisions. How did you choose your particular visual encodings and interaction techniques? What alternatives did you consider and how did you arrive at your ultimate choices?</li>
        <p>
            Regarding my 2nd chart, which is the custom visualization I made to examine the same AQI data but in another novel way, I used a variety of techniques to enhance the user experience and provide deeper insights into the data. This included experimenting with different chart types, such as bar charts and line graphs, to find the most effective way to represent the information. <br>
            To start, I used a brush interaction to allow users to zoom in and explore details within specific time intervals. During initial trials, I noticed that visualizations showing AQI readings across all stations became cluttered and difficult to interpret, especially when multiple datasets were overlaid. By implementing brushing, users can focus on smaller temporal segments, revealing 
            nuanced fluctuations that would otherwise be lost at the full-scale view. I considered adding separate small multiples for each station as an alternative, but ultimately chose brushing because it maintained context and continuity meaning that users could zoom into regions of interest without losing sight of the overall pattern.
            In terms of color and opacity, I carefully adjusted the value and saturation levels to guide the viewer’s attention toward the most important features. For example, the average AQI line was assigned a stronger color weight compared to the 10–90% area band, visually communicating its importance while ensuring that the variability region still provided useful context. I used blue and brown to distinguish the primary and secondary dataset stations, as these colors contrasted effectively without overwhelming the viewer or creating accessibility issues for color-blind users.
            Color was also central to the interactive AQI level toggles, where users could select a specific AQI range to highlight corresponding raw datapoints. This interaction helped reveal how frequently certain AQI thresholds were exceeded and how data distributions aligned with the official AQI categories, all without requiring a secondary chart. I considered static legends or separate histograms to show these relationships but found that toggling provided a more dynamic and intuitive experience, enabling users to directly compare filtered subsets of data within the same spatial and temporal context.
            Overall, these design decisions were chosen to balance clarity, engagement, and analytical depth, making the visualization both informative and exploratory.
        </p>
        <li>A clear description of the goals of your second visualization. Describe the question that you are enabling a user to answer. The question should be compelling and the solution should be focused on helping users achieve their goals.</li>
        <p>
            The main question this visualization helps answer is:
        </p>
            <blockquote>“How do air quality trends differ between two stations over time?”</blockquote>
        <p>
            To address this question, the visualization’s main goal is to help users compare air quality (AQI) averages between two monitoring stations in a clear and interactive way. Users can select a primary and secondary dataset and explore how their AQI levels fluctuate over time through the average AQI line and the 10–90% variability band. The brushing functionality enables users to zoom into specific time periods, revealing detailed trends and fluctuations that may not be visible at a high-level view.
            Additionally, the AQI level toggles allow users to filter and highlight raw data points based on official AQI categories, helping them see how often certain air quality thresholds are exceeded and how these patterns differ between the two stations. Together, these features support a detailed, side-by-side comparison that encourages users to identify patterns, seasonal shifts, and potential health implications tied to air quality differences.
            While I initially considered showing all stations at once, this approach made the chart visually cluttered and difficult to interpret. Limiting the comparison to two datasets at a time ensures that differences in monthly AQI levels remain clear and meaningful, helping users gain a deeper understanding of air quality trends between specific locations.
        </p>
        <li>An overview of your development process. Describe how the work was split among the team members. Include a commentary on the development process, including answers to the following questions: Roughly how much time did you spend developing your application (in people-hours)? What aspects took the most time? How did you use AI? Provide the prompts you used.</li>
        <p>
            The development process was completed individually, though I consulted with peers for feedback on my first draft of the chart. The process was done incrementally, beginning with Part 1, where I focused on building a functional base visualization of the AQI dataset. Using what I learned from that stage, I moved on to Part 2, where I improved interactivity by adding basic features such as zooming and tooltips. Once I had a rudimentary draft, I submitted it for peer review, where I received helpful feedback such as styling changes and how my Part 2 chart appeared too static and could benefit from more interactive components.<br>
            Taking this feedback into account, I used AI tools to brainstorm ways to make the chart more dynamic and insightful. One of the main prompts I used was:
        </p>
            <blockquote><i>“Based on this chart, I received feedback that it is not interactive enough or that the data formatting could be improved.<br> What are some ways I can increase interactivity or display the data in a new way on the chart?”</i></blockquote>
        <p>
            From the AI’s suggestions, I explored ideas such as adding toggleable data series, which inspired the AQI level toggles, and dynamic comparisons, which led me to focus on comparing two datasets at a time rather than overlaying all stations. These changes made the visualization more engaging and aligned it more closely with my guiding question about differences between stations.<br>
            In total, I spent roughly 16-18 hours developing the application. The most time-consuming aspects were debugging interactivity features (such as brushing and toggling) and fine-tuning color and layout to improve readability and visual balance. Overall, this was an iterative and feedback-driven process, where each stage built upon the previous one through experimentation and refinement, resulting in a more polished and interactive final visualization.
        </p>
    </ul>
</div>

<style>
    :global(*) {
        font-family: sans-serif;
    }

    .report-content {
        margin-top: 2rem;
        line-height: 1.6;
        font-size: 1rem;
    }

    .report-content p {
        margin-bottom: 2rem;
        color: #03015f;
    }

    .report-content blockquote {
        margin: 1rem 2rem;
        padding: 0.5rem 1rem;
        background-color: #f0f0f0;
        color: #03015f;
        border-left: 4px solid #ccc;
        font-style: italic;
    }
    
    button {
        padding: 0.5rem 1rem;
        border-radius: 4px;
        border: 1px solid #ccc;
        background-color: white;
        cursor: pointer;
        font-size: 1rem;
        margin-bottom: 2rem;
    }

    button:hover {
        background-color: #f0f0f0;
    }
</style>